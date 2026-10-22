# Demo 07: Stablecoin DEX

Students watch a payment app convert alphaUSD into betaUSD through Tempo's enshrined Stablecoin DEX on testnet: it quotes, swaps with a minimum, hits two real reverts (a minimum the book cannot meet, a token with no liquidity), then pays a betaUSD-only merchant in one atomic transaction. The lesson: a quote is only a preview. What protects the payer is the `minAmountOut`/`maxAmountIn` bound that the DEX enforces, and what protects the merchant is checking the final transfer in the receipt.

## Questions This Demo Answers

* Is the Stablecoin DEX the same thing as the Fee AMM? No. The DEX is the singleton precompile at `0xdec0…0000` that users and apps call to convert value against an orderbook. The Fee AMM lives in the FeeManager at `0xfeec…0000` and converts transaction fees at a fixed 0.9970 rate without anyone asking. In step 5's receipt, the swap legs go through `Stablecoin DEX` and the gas fee goes to `FeeManager`.
* How does alphaUSD reach betaUSD when there is no alphaUSD/betaUSD pair? Each TIP-20 declares exactly one `quoteToken()`. alphaUSD and betaUSD both point at pathUSD, so the only route is alphaUSD → pathUSD → betaUSD, and one `swapExactAmountIn` call walks both hops. Step 1 shows the hop-by-hop quotes landing on exactly the single quote, and in step 5's receipt the only pathUSD transfer is the gas fee: the hop never touches the payer's wallet.
* Can the app mark the invoice paid as soon as it submits the swap? No. A swap only converts the payer's own money: in step 2 the betaUSD goes back to the payer, not to the merchant. A submitted transaction can also revert (step 3, and the exercise). The script marks INV-1042 paid only after the receipt says `success` and contains a betaUSD `TransferWithMemo` to the merchant for exactly 250 with the invoice memo.
* Why set `minAmountOut` or `maxAmountIn` when I already have a quote? A quote is a view call against the book at one block. The swap fills later, against whatever the book is then, and the bound is the only price guarantee the DEX enforces. Quotes and fills can also simply differ: on testnet after the T12 upgrade (October 8, 2026), selling 100 alphaUSD quoted 99.980001 betaUSD and filled 99.99, and the exact-out buy in step 5 quoted 250.050006 alphaUSD but cost 248.533798. A bound the book cannot meet reverts with `InsufficientOutput` (exact in) or `MaxInputExceeded` (exact out), and nothing moves.
* What happens if there is not enough liquidity? The quote reverts before anyone signs anything. You get `PairDoesNotExist` when nobody has created a pair for the token, and `InsufficientLiquidity` when the pair exists but the book cannot fill the amount. A swap reverts the same way. The app should show "cannot convert" and not start the payment.
* Who bears the conversion risk? The payer. With an exact-out buy, the merchant receives exactly 250 betaUSD, and the payer's alphaUSD cost can move up to `maxAmountIn`. In the sample run the quote was 250.050006, the payer paid 248.533798, and the cap was 252.550506. If the price moves past the cap, the whole batch reverts and neither side moves any money.

## 1. Create The Project

```bash
mkdir tempo-demo-07-stablecoin-dex
cd tempo-demo-07-stablecoin-dex
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/stablecoinDex.ts`:

```ts
import {
  type Address,
  type BaseError,
  formatUnits,
  hexToString,
  isAddressEqual,
  parseEventLogs,
  parseUnits,
  stringToHex,
  zeroAddress,
} from "viem";
import { generatePrivateKey } from "viem/accounts";
import { Abis, Account, Actions, Addresses, createClient, ExecutionError, http, Tick } from "viem/tempo";

// Faucet stablecoins on Tempo testnet (all 6 decimals).
const pathUsd = "0x20c0000000000000000000000000000000000000";
const alphaUsd = "0x20c0000000000000000000000000000000000001";
const betaUsd = "0x20c0000000000000000000000000000000000002";
const explorer = "https://explore.testnet.tempo.xyz/tx/";

// The public testnet RPC allows a few dozen requests per second per IP, so a class on one network
// gets rate-limited (HTTP 429 or a JSON-RPC "rate limited" error). Those requests were never
// processed, so waiting and sending them again is safe.
async function fetchWithRetry(input: string | URL | Request, init?: RequestInit) {
  for (let attempt = 1; ; attempt++) {
    const response = await fetch(input, init);
    const limited = response.status === 429 || (await response.clone().text()).includes("rate limited");
    if (!limited || attempt === 40) return response;
    await new Promise((resolve) => setTimeout(resolve, 500 + Math.random() * 1_000));
  }
}

// Fresh throwaway accounts on every run. Fees are paid in pathUSD so that
// alphaUSD and betaUSD balances only move because of the DEX.
const payer = Account.fromSecp256k1(generatePrivateKey());
const merchant = Account.fromSecp256k1(generatePrivateKey());
const client = createClient({
  account: payer,
  testnet: true,
  feeToken: pathUsd,
  transport: http(undefined, { fetchFn: fetchWithRetry, timeout: 60_000 }),
});

const labels = new Map<string, string>([
  [zeroAddress, "none (root)"], [pathUsd, "pathUSD"], [alphaUsd, "alphaUSD"], [betaUsd, "betaUSD"],
  [Addresses.stablecoinDex, "Stablecoin DEX"], [Addresses.feeManager, "FeeManager"],
  [payer.address.toLowerCase(), "payer"], [merchant.address.toLowerCase(), "merchant"],
]);
const label = (address: string) => labels.get(address.toLowerCase()) ?? address;
const usd = (amount: bigint) => formatUnits(amount, 6);
const balanceOf = async (token: Address, account: Address = payer.address) =>
  (await client.token.getBalance({ token, account })).amount;

// Run an action that must revert with a specific error, and print the decoded error on one line.
async function expectRevert(what: string, expected: string, action: () => Promise<unknown>) {
  try {
    await action();
  } catch (error) {
    const decoded = ExecutionError.from(error as Error);
    if (decoded.errorName !== expected) throw error;
    console.log(`   ${what}: reverted with ${decoded.errorName} (${decoded.message})`);
    return;
  }
  throw new Error(`${what}: expected ${expected}, but it succeeded`);
}

// The books are shared with every other trader on testnet. If they change between our gas estimate
// and execution, the transaction can revert on chain (out of gas or past its price bound) and nothing
// moves except the fee. A real app quotes again and retries, so this demo does the same.
async function retryIfBookMoved<T>(run: () => Promise<T>): Promise<T> {
  for (let attempt = 1; ; attempt++) {
    try {
      return await run();
    } catch (error) {
      const reverted = (error as BaseError).walk?.((cause) => (cause as Error).name === "TransactionReceiptRevertedError");
      if (!reverted || attempt === 3) throw error;
      console.log("   the book changed before this executed, so it reverted on chain; trying again");
    }
  }
}

console.log(`Payer:    ${payer.address}`);
console.log(`Merchant: ${merchant.address}`);
await client.faucet.fundSync({ account: payer.address, timeout: 60_000 });
console.log("Faucet funded the payer with 1,000,000 of each testnet stablecoin.");

console.log("\n1. Routing tree, read from each token's quoteToken()");
for (const token of [alphaUsd, betaUsd, pathUsd] as const) {
  const quoteToken = await client.readContract({ address: token, abi: Abis.tip20, functionName: "quoteToken" });
  console.log(`   ${label(token).padEnd(8)} -> quote token ${label(quoteToken)}`);
}
const amountIn = parseUnits("100", 6);
const blockNumber = await client.getBlockNumber(); // read every quote from the same block
const hop1 = await client.dex.getSellQuote({ tokenIn: alphaUsd, tokenOut: pathUsd, amountIn, blockNumber });
const hop2 = await client.dex.getSellQuote({ tokenIn: pathUsd, tokenOut: betaUsd, amountIn: hop1, blockNumber });
const direct = await client.dex.getSellQuote({ tokenIn: alphaUsd, tokenOut: betaUsd, amountIn, blockNumber });
console.log(`   hop by hop: 100 alphaUSD -> ${usd(hop1)} pathUSD -> ${usd(hop2)} betaUSD`);
console.log(`   one quote:  100 alphaUSD -> ${usd(direct)} betaUSD (the DEX walks both hops)`);

console.log("\n2. Quote first, then sell 100 alphaUSD for betaUSD with a minimum");
const slippageBps = 100n; // 1% tolerance: testnet books are shared with every student and bot
const alphaBefore = await balanceOf(alphaUsd);
const betaBefore = await balanceOf(betaUsd);
let quote = 0n;
const swap = await retryIfBookMoved(async () => {
  quote = await client.dex.getSellQuote({ tokenIn: alphaUsd, tokenOut: betaUsd, amountIn });
  const minAmountOut = quote - (quote * slippageBps) / 10_000n;
  console.log(`   quote ${usd(quote)} betaUSD, minAmountOut ${usd(minAmountOut)}`);
  return (await client.dex.sellSync({ tokenIn: alphaUsd, tokenOut: betaUsd, amountIn, minAmountOut })).receipt;
});
const alphaAfter = await balanceOf(alphaUsd);
const betaAfter = await balanceOf(betaUsd);
console.log(`   swap ${explorer}${swap.transactionHash}`);
console.log(`   alphaUSD ${usd(alphaBefore)} -> ${usd(alphaAfter)}`);
console.log(`   betaUSD  ${usd(betaBefore)} -> ${usd(betaAfter)}`);
console.log(`   received ${usd(betaAfter - betaBefore)} betaUSD (quote said ${usd(quote)})`);

console.log("\n3. Same swap, but the minimum is 1% above what the book gives");
const newQuote = await client.dex.getSellQuote({ tokenIn: alphaUsd, tokenOut: betaUsd, amountIn });
const tooHigh = newQuote + newQuote / 100n;
console.log(`   quote ${usd(newQuote)} betaUSD, minAmountOut ${usd(tooHigh)}`);
await expectRevert("sell", "InsufficientOutput", () =>
  client.dex.sellSync({ tokenIn: alphaUsd, tokenOut: betaUsd, amountIn, minAmountOut: tooHigh }),
);
if ((await balanceOf(alphaUsd)) !== alphaAfter || (await balanceOf(betaUsd)) !== betaAfter) {
  throw new Error("Balances moved after a rejected swap");
}
console.log("   balances unchanged: the node rejected the swap while filling it, nothing was broadcast");

console.log("\n4. Liquidity too low: a brand-new USD stablecoin nobody trades yet");
const stamp = Date.now().toString(36).toUpperCase();
const created = await client.token.createSync({
  name: `Demo07 USD ${stamp}`,
  symbol: `D7${stamp}`,
  currency: "USD", // quoteToken defaults to pathUSD
});
const newUsd = created.token;
console.log(`   created ${created.symbol} ${newUsd}, quote token ${label(created.quoteToken)}`);
await expectRevert("quote before createPair", "PairDoesNotExist", () =>
  client.dex.getBuyQuote({ tokenIn: alphaUsd, tokenOut: newUsd, amountOut: amountIn }),
);
const { receipt: pair } = await client.dex.createPairSync({ base: newUsd });
console.log(`   createPair is a call on the DEX precompile, no contract deployed:`);
console.log(`   ${explorer}${pair.transactionHash}`);
await expectRevert("quote after createPair", "InsufficientLiquidity", () =>
  client.dex.getBuyQuote({ tokenIn: alphaUsd, tokenOut: newUsd, amountOut: amountIn }),
);

console.log("\n5. Pay a merchant who only accepts betaUSD, from alphaUSD, in ONE transaction");
const invoice = parseUnits("250", 6);
const memo = stringToHex("INV-1042", { size: 32 });
const receipt = await retryIfBookMoved(async () => {
  const cost = await client.dex.getBuyQuote({ tokenIn: alphaUsd, tokenOut: betaUsd, amountOut: invoice });
  const maxAmountIn = cost + (cost * slippageBps) / 10_000n;
  console.log(`   invoice 250 betaUSD, quoted cost ${usd(cost)} alphaUSD, maxAmountIn ${usd(maxAmountIn)}`);
  const calls = [
    Actions.dex.buy.call({ tokenIn: alphaUsd, tokenOut: betaUsd, amountOut: invoice, maxAmountIn }),
    Actions.token.transfer.call({ token: betaUsd, to: merchant.address, amount: invoice, memo }),
  ];
  return client.sendTransactionSync({ calls, throwOnReceiptRevert: true });
}).catch((error: Error) => {
  console.log(`   batch reverted with ${ExecutionError.from(error).errorName} -> invoice NOT paid`);
  process.exit(1);
});
console.log(`   batch ${explorer}${receipt.transactionHash}`);
console.log("   transfers in the receipt:");
for (const log of parseEventLogs({ abi: Abis.tip20, eventName: "Transfer", logs: receipt.logs })) {
  const amount = usd(log.args.amount).padStart(10);
  console.log(`     ${label(log.address).padEnd(8)} ${amount}  ${label(log.args.from)} -> ${label(log.args.to)}`);
}
const allowance = await client.token.getAllowance({
  token: alphaUsd,
  account: payer.address,
  spender: Addresses.stablecoinDex,
});
console.log(`   payer's alphaUSD allowance for the DEX: ${allowance.formatted} (no approve was needed)`);

// Merchant side: mark the invoice paid only if the final transfer has the right token, recipient, amount, memo.
const payment = parseEventLogs({ abi: Abis.tip20, eventName: "TransferWithMemo", logs: receipt.logs }).find(
  (log) => isAddressEqual(log.address, betaUsd) && isAddressEqual(log.args.to, merchant.address),
);
if (receipt.status !== "success" || payment?.args.amount !== invoice || payment.args.memo !== memo) {
  throw new Error("Invoice is NOT paid");
}
const reference = hexToString(payment.args.memo, { size: 32 });
console.log(`   merchant got ${usd(payment.args.amount)} betaUSD with memo "${reference}" -> invoice PAID`);

console.log("\n6. Maker side: rest a limit order on the book, then take it back");
const tick = Tick.fromPrice("0.999"); // 1 bp grid: price = 1 + tick / 100_000
const order = await retryIfBookMoved(() =>
  client.dex.placeSync({ token: betaUsd, type: "buy", amount: parseUnits("100", 6), tick }),
);
// Read our own order: since T12 a price level's total is summed from its orders on every call,
// which can run out of gas on a crowded level, while one order is a cheap lookup.
const resting = await client.dex.getOrder({ orderId: order.orderId });
const ahead = resting.prev === 0n ? "nothing ahead of it" : `behind order #${resting.prev}`;
const position = resting.next === 0n ? "last in that price's FIFO queue" : "queued";
console.log(`   order #${order.orderId}: bid for 100 betaUSD at ${Tick.toPrice(order.tick)} pathUSD (tick ${order.tick})`);
console.log(`   resting ${usd(resting.remaining)} betaUSD, ${ahead}: ours is ${position}`);
await retryIfBookMoved(() => client.dex.cancelSync({ orderId: order.orderId }));
const refund = await client.dex.getBalance({ account: payer.address, token: pathUsd });
console.log(`   cancelled: the ${usd(refund)} pathUSD escrow went back to the payer's internal DEX balance`);
await client.dex.withdrawSync({ token: pathUsd, amount: refund });
const leftOnDex = await client.dex.getBalance({ account: payer.address, token: pathUsd });
console.log(`   withdrawn to the wallet; internal DEX balance is now ${usd(leftOnDex)}`);

console.log("\nDone. Every quote, transfer and revert above came from Tempo testnet.");
```

## 3. Run The Demo

```bash
npx tsx src/stablecoinDex.ts
```

Output from a real run (addresses and hashes will differ):

```text
Payer:    0x558aa85Cf2540EAD3b095C3447E81aDB93401345
Merchant: 0x6f2157a6906aEd96FDba44452854FEaBd14CB758
Faucet funded the payer with 1,000,000 of each testnet stablecoin.

1. Routing tree, read from each token's quoteToken()
   alphaUSD -> quote token pathUSD
   betaUSD  -> quote token pathUSD
   pathUSD  -> quote token none (root)
   hop by hop: 100 alphaUSD -> 99.99 pathUSD -> 99.980001 betaUSD
   one quote:  100 alphaUSD -> 99.980001 betaUSD (the DEX walks both hops)

2. Quote first, then sell 100 alphaUSD for betaUSD with a minimum
   quote 99.980001 betaUSD, minAmountOut 98.980201
   swap https://explore.testnet.tempo.xyz/tx/0xfc46db178e21042bac8a7929080cc9a78c41211165521962ede0d97931a65aeb
   alphaUSD 1000000 -> 999900
   betaUSD  1000000 -> 1000099.99
   received 99.99 betaUSD (quote said 99.980001)

3. Same swap, but the minimum is 1% above what the book gives
   quote 99.980001 betaUSD, minAmountOut 100.979801
   sell: reverted with InsufficientOutput (The output amount is below the slippage minimum — try increasing slippage tolerance.)
   balances unchanged: the node rejected the swap while filling it, nothing was broadcast

4. Liquidity too low: a brand-new USD stablecoin nobody trades yet
   created D7MUZY5M0M 0x20C000000000000000000000222625060002Dd09, quote token pathUSD
   quote before createPair: reverted with PairDoesNotExist (No trading pair exists for these tokens.)
   createPair is a call on the DEX precompile, no contract deployed:
   https://explore.testnet.tempo.xyz/tx/0x4defea3665eea3edb223fd97f4ab22e675e0a2b674299aa52b7fca8a5735c847
   quote after createPair: reverted with InsufficientLiquidity (Not enough liquidity in the order book to fill this trade.)

5. Pay a merchant who only accepts betaUSD, from alphaUSD, in ONE transaction
   invoice 250 betaUSD, quoted cost 250.050006 alphaUSD, maxAmountIn 252.550506
   batch https://explore.testnet.tempo.xyz/tx/0xbb32009c9eeb05f50937d4148b5f699ff8c970505d359646a902d4375a42b815
   transfers in the receipt:
     alphaUSD 248.533798  payer -> Stablecoin DEX
     betaUSD         250  Stablecoin DEX -> payer
     betaUSD         250  payer -> merchant
     pathUSD    0.000258  payer -> FeeManager
   payer's alphaUSD allowance for the DEX: 0 (no approve was needed)
   merchant got 250 betaUSD with memo "INV-1042" -> invoice PAID

6. Maker side: rest a limit order on the book, then take it back
   order #18429578: bid for 100 betaUSD at 0.999 pathUSD (tick -100)
   resting 100 betaUSD, behind order #18429573: ours is last in that price's FIFO queue
   cancelled: the 99.9 pathUSD escrow went back to the payer's internal DEX balance
   withdrawn to the wallet; internal DEX balance is now 0

Done. Every quote, transfer and revert above came from Tempo testnet.
```

Walk through the output in this order:

* **Setup.** Every run creates two fresh accounts, and the faucet gives the payer 1,000,000 each of pathUSD, alphaUSD, betaUSD and thetaUSD. The client sets `feeToken: pathUsd`, so alphaUSD and betaUSD balances only move because of the DEX. Without it, a transaction with a single swap call pays its fee in `tokenIn` (alphaUSD here). `fetchWithRetry` is there because the public RPC rate-limits bursts (HTTP 429 or a "rate limited" error), and viem never retries a transaction submission on its own. A classroom on one Wi-Fi network will hit this. `retryIfBookMoved` handles a second shared-state problem: if someone else's trade changes the book between your gas estimate and execution, a swap or order can revert on chain (we saw out-of-gas reverts when many copies ran at once), so the helper quotes again and retries, as a real app would. Rejections at the quote or fill stage, like step 3, are never retried.
* **Step 1, routing tree.** These are the raw `quoteToken()` values: alphaUSD and betaUSD point at pathUSD, and pathUSD points at the zero address because it is the root. All three quotes read the same block, so the hop-by-hop result (100 alphaUSD → pathUSD → betaUSD) matches the single alphaUSD → betaUSD quote exactly. The DEX finds the unique path itself.
* **Step 2, quote then swap.** `getSellQuote` is a free view call. `sellSync` sends `swapExactAmountIn` with `minAmountOut` set to the quote minus 1%. There is no `approve`: since the T5 upgrade, TIP-20 lets listed precompiles such as the DEX pull tokens without an allowance. Compare "received" with "quote said". Since the T12 upgrade (testnet, October 8, 2026), quotes use the same per-order matching as swaps, and in this run the fill beat the quote by 0.01 betaUSD. Before T12, a stale price level on the alphaUSD book made testnet quotes about 0.5% more optimistic than fills. Either way, the bound is what protects the payer: a quote is a preview, not a promise.
* **Step 3, bound too tight.** `minAmountOut` is set 1% above the quote, so the DEX reverts with `InsufficientOutput`. viem asks the node to fill and simulate the transaction before signing it. The precompile reverts during that simulation, so nothing is broadcast and no fee is paid, and the script re-reads both balances to prove it. If the price moved after broadcast, the same check would revert the transaction inside the block.
* **Step 4, no liquidity.** The TIP-20 factory creates a new USD token, whose quote token defaults to pathUSD. A quote before any pair exists reverts with `PairDoesNotExist`. `createPair` is one call on the DEX singleton, with no contract deployed per pair. The pair starts with an empty book, so the quote now reverts with `InsufficientLiquidity`. Either error means the app must not start the payment.
* **Step 5, atomic payment.** This is one Tempo Transaction with two `calls`. `Actions.dex.buy.call` is `swapExactAmountOut` for exactly 250 betaUSD, with `maxAmountIn` set to the quote plus 1%. `Actions.token.transfer.call` with a `memo` becomes `transferWithMemo`. Read the transfers table from top to bottom: the DEX pulls alphaUSD from the payer, swap output always goes to the caller (DEX → payer), the payer forwards exactly 250 betaUSD to the merchant, and the fee leaves in pathUSD to the FeeManager. The pathUSD hop happens inside the DEX. The alphaUSD allowance is still 0. The merchant check reads `TransferWithMemo` from the receipt and needs the right token, recipient, amount and memo before it prints PAID.
* **Step 6, maker side (where liquidity comes from).** This is a limit bid for 100 betaUSD at 0.999, which is tick −100. Prices sit on a 1 bp grid: `PRICE_SCALE` is 100_000, ticks are multiples of 10, the range is ±2000 ticks (±2%), and the minimum order is 100 tokens. The order joins the end of the FIFO queue at its price, which is price-time priority: `getOrder` shows the order ahead of ours and nothing behind it. The script reads its own order rather than the whole price level because, since T12, a level's total is summed from its orders on every read, and on a crowded level like this one `getTickLevel` runs out of gas. Placing it escrowed 99.9 pathUSD from the wallet. Cancelling refunds that escrow to the payer's internal DEX balance, not to the wallet, and `withdraw` moves it out. Swaps spend that internal balance before pulling the shortfall from the wallet.

## 4. Student Exercise

In step 5, change `const maxAmountIn = cost + (cost * slippageBps) / 10_000n;` to `const maxAmountIn = cost - cost / 100n;`, which means the payer will pay at most 1% less than the quote. Run the script again. Steps 1 to 4 print as before. Step 5 prints `batch reverted with MaxInputExceeded -> invoice NOT paid` and the script exits with code 1. Look up the merchant's address on the explorer (`https://explore.testnet.tempo.xyz/address/<merchant>`): no betaUSD arrived. The payer's alphaUSD shows only the 100 sold in step 2, because the swap call and the transfer call succeed or fail together. Don't use `cost - 1n` for this exercise: on testnet the book can move in the payer's favor by more than one micro-unit between quote and fill. After T12 the exact-out quote ran about 0.6% above the real cost (250.05 quoted, 248.53 paid), which is why the exercise goes a full 1% under the quote.

## Production Takeaway

Quote right before you submit, but rely on the bound: give every swap a `minAmountOut` or `maxAmountIn` sized for that pair, and treat a reverting quote as "cannot convert". When a conversion pays someone, put the exact-out buy and the transfer in one Tempo Transaction. Mark the invoice paid only from a successful receipt whose `TransferWithMemo` matches token, recipient, amount and memo. Set an explicit fee token, and use your own RPC endpoint rather than the rate-limited public one.

## References

* [Stablecoin DEX Specification](https://tempo.xyz/developers/docs/protocol/exchange/spec)
* [Exchanging stablecoins: exchange vs. Fee AMM](https://tempo.xyz/developers/docs/protocol/exchange)
* [Executing swaps | Protocol reference](https://tempo.xyz/developers/docs/protocol/exchange/executing-swaps)
* [Understanding DEX balances on Tempo](https://tempo.xyz/developers/docs/protocol/exchange/exchange-balance)
* [pathUSD on Tempo Mainnet and quote tokens](https://tempo.xyz/developers/docs/protocol/exchange/quote-tokens)
* [Fee AMM overview: converting stablecoin fees](https://tempo.xyz/developers/docs/protocol/fees/fee-amm)
* [Fee specification: fee token preferences](https://tempo.xyz/developers/docs/protocol/fees/spec-fee)
* [TIP-20 Tokens Specification (memos, implicit approvals)](https://tempo.xyz/developers/docs/protocol/tip20/spec)
* [T5 Network Upgrade (implicit approvals)](https://tempo.xyz/developers/docs/protocol/upgrades/t5)
* [T12 Network Upgrade (quotes aligned with swaps)](https://tempo.xyz/developers/docs/protocol/upgrades/t12)
* [Using Tempo Transactions: batch calls](https://tempo.xyz/developers/docs/guide/tempo-transaction)
* [Attach a Transfer Memo](https://tempo.xyz/developers/docs/guide/payments/transfer-memos)
* [viem: Swap Stablecoins](https://viem.sh/tempo/guides/stablecoin-exchange/swap)
* [viem: dex.sell](https://viem.sh/tempo/actions/dex.sell) and [dex.buy](https://viem.sh/tempo/actions/dex.buy)
