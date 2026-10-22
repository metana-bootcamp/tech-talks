# Demo 10: Fee AMM

Students watch Tempo charge gas in stablecoins on testnet, then look at the machine behind it: the Fee AMM. The same alphaUSD payment pays its fee in alphaUSD, betaUSD or pathUSD depending on what the transaction and the account ask for, and the block's proposer is paid in the token it wants through a fixed-price swap. Then a brand-new stablecoin tries to pay its own fee, and the node refuses the transaction. The issuer deposits 100 pathUSD into the coin's Fee AMM pool, the same payment goes through, and the pool's reserves move by exactly the fee in and 0.997 of the fee out, rounded down. The lesson: any USD stablecoin can pay gas on Tempo, but only if a Fee AMM pool can convert it into the token the validator wants, so launching a stablecoin includes seeding and watching that pool.

## Questions This Demo Answers

* Is the Fee AMM the same thing as the Stablecoin DEX? No. The DEX is the precompile (a contract built into the node) at `0xdec0…0000` (demo 07): an orderbook that users and apps call to convert value at market prices. The Fee AMM lives inside the FeeManager precompile at `0xfeec…0000` and only converts transaction fees, at a fixed price of 0.9970. Nobody calls it to make a swap: the protocol runs the swap while it charges the fee. You call it only to add or remove liquidity (`mint`, `burn`) or to refill a pool (`rebalanceSwap`). This demo never touches the DEX.
* Which token pays the fee when I don't pick one? The protocol checks, in order: the transaction's `feeToken`, the account's default set with `setUserToken`, the token itself when the transaction is a TIP-20 `transfer` (or the `tokenIn` of a DEX swap), and finally pathUSD. Step 1 shows three of these in real receipts: a plain alphaUSD transfer pays in alphaUSD, the same transfer with `feeToken: betaUSD` pays in betaUSD, and an `approve`, which is not a transfer, falls back to pathUSD. Step 4 shows the account default beating the token being moved. The receipt's `feeToken` field says which token paid, and the fee also appears in the receipt as a TIP-20 `Transfer` to the FeeManager.
* Who gets the fee, and in which token? The block's proposer, in the token it chose with `setValidatorToken`, or pathUSD if it never chose one. On October 8, 2026, most testnet proposers wanted pathUSD and one (`0x7a8d…5a65`) wanted betaUSD: 194 of 200 sampled blocks wanted pathUSD, and in some stretches the betaUSD proposer made 1 block in 6. Tempo's fee spec says Moderato validators expect alphaUSD, but the chain disagreed on the day we tested. When the fee token differs from the proposer's token, the FeeManager swaps the fee in the Fee AMM pool for that pair: the proposer gets 0.997 of the fee, rounded down to a whole micro-unit, and the pool keeps the rest.
* Why can't a brand-new stablecoin pay its own gas? Because the protocol must pay the proposer in the proposer's token, and the new coin's pools are empty. Before it accepts a transaction, the node checks that a pool can cover the worst-case fee (gas limit × max fee per gas), either directly or, since T5, in two hops through the coin's quote token (the token it is priced against, pathUSD by default). The block producer checks again when it runs the transaction. In step 3 the node answers `insufficient liquidity in FeeAMM pool to swap fee tokens for user token 0x… (required: 419)`, where 419 is that worst-case fee in micro-units. The transaction never reaches a block, so no fee is charged and the nonce is not used.
* How does the issuer fix it, and what does it cost? Deposit a validator's token (pathUSD here) into the pool with `amm.mint`. You deposit only the validator side; the other side starts at zero and fills up with fees. A pool's first deposit must be at least 2,002 micro-units (0.002002 tokens), because the FeeManager creates LP units (liquidity-provider shares) worth half the deposit and locks 1,000 of them forever (`MIN_LIQUIDITY`), which costs about $0.002. One pool against pathUSD is enough: a proposer that wants betaUSD is paid through two hops, new coin → pathUSD → betaUSD, and each hop takes its own 0.3%.
* Who earns the 0.3%? The liquidity providers (LPs), the accounts that deposited into the pool and hold its LP units. Each fee swap puts 1 unit of the user's token into the pool and takes out 0.997 of the validator's token: in step 3 the pool took in 169 units and paid out 168. Over time the pool fills up with the user's token and runs low on the validator's token. Anyone can then refill it with `rebalanceSwap`, which pays 0.9985 of the validator's token for each unit of the user's token taken out, also at a fixed price.

## 1. Create The Project

```bash
mkdir tempo-demo-10-fee-amm
cd tempo-demo-10-fee-amm
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates a throwaway testnet account and funds it from the Tempo faucet on every run.

## 2. Write The Script

Create `src/feeAmm.ts`:

```ts
import { type Address, type BaseError, formatUnits, isAddressEqual, type Log, parseEventLogs, parseUnits } from "viem";
import { generatePrivateKey } from "viem/accounts";
import { Abis, Account, Addresses, createClient, http } from "viem/tempo";

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
const transport = http(undefined, { fetchFn: fetchWithRetry, timeout: 60_000 });

// A fresh throwaway account on every run. The client sets NO fee token on purpose:
// this demo is about which token the protocol charges when nobody picks one.
const payer = Account.fromSecp256k1(generatePrivateKey());
const merchant = Account.fromSecp256k1(generatePrivateKey()).address;
const client = createClient({ account: payer, testnet: true, transport });

const symbols = new Map<string, string>([[pathUsd, "pathUSD"], [alphaUsd, "alphaUSD"], [betaUsd, "betaUSD"]]);
const symbol = (token: string) => symbols.get(token.toLowerCase()) ?? token;
const usd = (amount: bigint) => formatUnits(amount, 6);

// The fee token a block's proposer wants. The FeeManager answers pathUSD for proposers that never chose.
const wanted = new Map<Address, Address>();
async function wantedBy(blockNumber: bigint) {
  const { miner } = await client.getBlock({ blockNumber });
  if (!wanted.has(miner)) wanted.set(miner, (await client.fee.getValidatorToken({ validator: miner }))!.address);
  return wanted.get(miner)!;
}

// Read the fee from a receipt: which token paid, how much, and what the block's proposer wanted.
type Receipt = { blockNumber: bigint; gasUsed: bigint; effectiveGasPrice: bigint; logs: Log[]; feeToken?: Address };
async function showFee(label: string, receipt: Receipt) {
  const feeToken = receipt.feeToken!;
  const fee = (receipt.gasUsed * receipt.effectiveGasPrice + 10n ** 12n - 1n) / 10n ** 12n; // 1e-18 USD -> 1e-6 USD, rounded up
  const transfers = parseEventLogs({ abi: Abis.tip20, eventName: "Transfer", logs: receipt.logs });
  const charged = (log: (typeof transfers)[number]) =>
    isAddressEqual(log.address, feeToken) && isAddressEqual(log.args.to, Addresses.feeManager) && log.args.amount === fee;
  if (!transfers.some(charged)) throw new Error(`${label}: no ${fee}-unit fee transfer to the FeeManager in the receipt`);
  const validatorToken = await wantedBy(receipt.blockNumber);
  console.log(`   ${label.padEnd(38)} fee ${usd(fee)} ${symbol(feeToken).padEnd(11)} proposer wants ${symbol(validatorToken)}`);
  return { fee, feeToken, validatorToken };
}

console.log("Demo 10: the Fee AMM on Tempo testnet (Moderato, chain 42431)\n");
console.log(`Payer ${payer.address}`);
await client.faucet.fundSync({ account: payer.address, timeout: 60_000 });
console.log("Faucet funded it with 1,000,000 each of pathUSD, alphaUSD, betaUSD and thetaUSD.");

console.log("\n1. Which token pays the fee? Each line is read from a real receipt");
const sendAlpha = async (feeToken?: Address) =>
  (await client.token.transferSync({ token: alphaUsd, to: merchant, amount: parseUnits("1", 6), feeToken })).receipt;
const approve = async () => (await client.token.approveSync({ token: alphaUsd, spender: merchant, amount: 1n })).receipt;
const fees = [
  await showFee("send 1 alphaUSD, nothing set", await sendAlpha()),
  await showFee("send 1 alphaUSD, feeToken: betaUSD", await sendAlpha(betaUsd)),
  await showFee("approve (not a transfer), nothing set", await approve()),
];

console.log(`\n2. Behind it: the Fee AMM inside the FeeManager precompile ${Addresses.feeManager}`);
const latest = await client.getBlockNumber();
const mix = new Map<string, number>();
for (let block = latest - 19n; block <= latest; block++) {
  const token = symbol(await wantedBy(block));
  mix.set(token, (mix.get(token) ?? 0) + 1);
}
console.log(`   fee token wanted by the proposers of the last 20 blocks: ${[...mix].map(([t, n]) => `${t} ${n}`).join(", ")}`);
const read = (functionName: "M" | "N" | "SCALE" | "MIN_LIQUIDITY") =>
  client.readContract({ address: Addresses.feeManager, abi: Abis.feeAmm, functionName });
const [M, N, SCALE, MIN_LIQUIDITY] = [await read("M"), await read("N"), await read("SCALE"), await read("MIN_LIQUIDITY")];
console.log(`   fixed prices in the contract: a fee swap pays out ${M}/${SCALE} per unit paid, a rebalance costs ${N}/${SCALE}`);
for (const { feeToken, validatorToken } of fees) {
  if (isAddressEqual(feeToken, validatorToken)) continue; // same token: nothing to convert
  const pool = await client.amm.getPool({ userToken: feeToken, validatorToken });
  const reserves = `${usd(pool.reserveUserToken)} ${symbol(feeToken)} / ${usd(pool.reserveValidatorToken)} ${symbol(validatorToken)}`;
  console.log(`   pool ${symbol(feeToken)} -> ${symbol(validatorToken)} swapped a fee above; reserves ${reserves}`);
}

console.log("\n3. A brand-new stablecoin cannot pay fees until someone seeds its Fee AMM pool");
const stamp = Date.now().toString(36).toUpperCase();
const created = await client.token.createSync({ name: `Demo10 USD ${stamp}`, symbol: `D10${stamp}`, currency: "USD" });
const newUsd = created.token; // its quote token defaults to pathUSD
const sym = created.symbol;
symbols.set(newUsd.toLowerCase(), sym);
await client.token.grantRolesSync({ token: newUsd, roles: ["issuer"], to: payer.address });
await client.token.mintSync({ token: newUsd, to: payer.address, amount: parseUnits("1000", 6) });
console.log(`   created ${sym} ${newUsd} (quote token ${symbol(created.quoteToken)}), minted 1000 to the payer`);
const payNew = async () => (await client.token.transferSync({ token: newUsd, to: merchant, amount: parseUnits("10", 6) })).receipt;
const refused = await payNew().then(() => undefined, (error: BaseError) => error);
if (!refused || !/insufficient ?liquidity/i.test(refused.details)) throw refused ?? new Error("expected a rejection");
console.log(`   send 10 ${sym}, nothing set, so the fee is charged in ${sym}: REJECTED by the node`);
console.log(`     "${refused.details}"`);
const { amount: balance } = await client.token.getBalance({ token: newUsd, account: payer.address });
console.log(`   balance still ${usd(balance)} ${sym}: refused before it reached a block, so no fee was charged`);
const seed = await client.amm.mintSync({
  userTokenAddress: newUsd, // the token users want to pay fees in
  validatorTokenAddress: pathUsd, // the token proposers are paid in: you deposit only this side
  validatorTokenAmount: parseUnits("100", 6),
  to: payer.address, // receives the LP units
});
const seeded = await client.amm.getPool({ userToken: newUsd, validatorToken: pathUsd });
console.log(`   seed the ${sym} -> pathUSD pool with ${usd(seed.amountValidatorToken)} pathUSD: ${explorer}${seed.receipt.transactionHash}`);
console.log(`     reserves ${usd(seeded.reserveUserToken)} ${sym} / ${usd(seeded.reserveValidatorToken)} pathUSD; ` +
  `LP units: ${seed.liquidity} for the payer + ${seeded.totalSupply - seed.liquidity} locked forever (MIN_LIQUIDITY ${MIN_LIQUIDITY})`);
const receipt = await payNew();
console.log(`   send 10 ${sym} again: ${explorer}${receipt.transactionHash}`);
const { fee, validatorToken } = await showFee(`  ${sym} paid its own fee`, receipt);
const pool = (blockNumber: bigint) => client.amm.getPool({ userToken: newUsd, validatorToken: pathUsd, blockNumber });
const [before, after] = [await pool(receipt.blockNumber - 1n), await pool(receipt.blockNumber)];
const paidIn = after.reserveUserToken - before.reserveUserToken;
const paidOut = before.reserveValidatorToken - after.reserveValidatorToken;
const route = isAddressEqual(validatorToken, pathUsd) ? "one hop" : `then pathUSD -> ${symbol(validatorToken)} in a shared pool`;
console.log(`     pool in that block: +${paidIn} ${sym} units in, -${paidOut} pathUSD units out (${route})`);
if (paidIn !== fee || paidOut !== (fee * M) / SCALE) throw new Error("the pool moved by more than our own fee");
console.log(`     ${paidOut} = floor(${fee} x ${M} / ${SCALE}): the fixed rate, and nothing else touched this pool`);

console.log(`\n4. Make ${sym} the payer's default fee token: it now wins over the token being moved`);
await showFee(`setUserToken(${sym})`, (await client.fee.setUserTokenSync({ token: newUsd })).receipt);
await showFee("send 1 alphaUSD, nothing set", await sendAlpha());
await showFee("approve (not a transfer), nothing set", await approve());
const end = await client.amm.getPool({ userToken: newUsd, validatorToken: pathUsd });
console.log(`   pool ${sym} -> pathUSD now: ${usd(end.reserveUserToken)} ${sym} collected, ${usd(end.reserveValidatorToken)} pathUSD to pay out`);

console.log("\nDone. Every fee, rejection and reserve above came from Tempo testnet.");
```

## 3. Run The Demo

```bash
npx tsx src/feeAmm.ts
```

Output from a real run on October 8, 2026, after T12 went live on testnet. It took about 12 seconds. Addresses, hashes and the "proposer wants" column change on every run:

```text
Demo 10: the Fee AMM on Tempo testnet (Moderato, chain 42431)

Payer 0x2795965beB2c64EF4b87f7c65B587D79E0ca0E94
Faucet funded it with 1,000,000 each of pathUSD, alphaUSD, betaUSD and thetaUSD.

1. Which token pays the fee? Each line is read from a real receipt
   send 1 alphaUSD, nothing set           fee 0.000319 alphaUSD    proposer wants pathUSD
   send 1 alphaUSD, feeToken: betaUSD     fee 0.000024 betaUSD     proposer wants pathUSD
   approve (not a transfer), nothing set  fee 0.000167 pathUSD     proposer wants pathUSD

2. Behind it: the Fee AMM inside the FeeManager precompile 0xfeec000000000000000000000000000000000000
   fee token wanted by the proposers of the last 20 blocks: pathUSD 18, betaUSD 2
   fixed prices in the contract: a fee swap pays out 9970/10000 per unit paid, a rebalance costs 9985/10000
   pool alphaUSD -> pathUSD swapped a fee above; reserves 560814955.715221 alphaUSD / 2006968068685.278995 pathUSD
   pool betaUSD -> pathUSD swapped a fee above; reserves 491233927.146845 betaUSD / 1006920767026.245783 pathUSD

3. A brand-new stablecoin cannot pay fees until someone seeds its Fee AMM pool
   created D10MUZYWIEK 0x20C000000000000000000000f730C6AD97fAf0fF (quote token pathUSD), minted 1000 to the payer
   send 10 D10MUZYWIEK, nothing set, so the fee is charged in D10MUZYWIEK: REJECTED by the node
     "insufficient liquidity in FeeAMM pool to swap fee tokens for user token 0x20C000000000000000000000f730C6AD97fAf0fF (required: 419)"
   balance still 1000 D10MUZYWIEK: refused before it reached a block, so no fee was charged
   seed the D10MUZYWIEK -> pathUSD pool with 100 pathUSD: https://explore.testnet.tempo.xyz/tx/0x5ff23ff2d269e5d3c05e8e6859b8b7582eccc36e5db9806a225b768143b0fbcb
     reserves 0 D10MUZYWIEK / 100 pathUSD; LP units: 49999000 for the payer + 1000 locked forever (MIN_LIQUIDITY 1000)
   send 10 D10MUZYWIEK again: https://explore.testnet.tempo.xyz/tx/0x2c71700bb0ceb75f6f067c5c1420a894518249f80841d2b5aef46ea25d86e165
     D10MUZYWIEK paid its own fee         fee 0.000169 D10MUZYWIEK proposer wants pathUSD
     pool in that block: +169 D10MUZYWIEK units in, -168 pathUSD units out (one hop)
     168 = floor(169 x 9970 / 10000): the fixed rate, and nothing else touched this pool

4. Make D10MUZYWIEK the payer's default fee token: it now wins over the token being moved
   setUserToken(D10MUZYWIEK)              fee 0.000167 D10MUZYWIEK proposer wants pathUSD
   send 1 alphaUSD, nothing set           fee 0.000024 D10MUZYWIEK proposer wants betaUSD
   approve (not a transfer), nothing set  fee 0.000016 D10MUZYWIEK proposer wants pathUSD
   pool D10MUZYWIEK -> pathUSD now: 0.000376 D10MUZYWIEK collected, 99.999628 pathUSD to pay out

Done. Every fee, rejection and reserve above came from Tempo testnet.
```

Walk through the output in this order:

* **Setup.** One fresh account, funded by the faucet with 1,000,000 of each testnet stablecoin. The client sets no fee token on purpose, so you see the protocol's own choice. `fetchWithRetry` waits and resends when the public RPC rate-limits a classroom (HTTP 429 or a "rate limited" error). `showFee` reads every fee from its receipt. Gas prices are quoted in attodollars (10^-18 USD) and a token unit is a microdollar (10^-6 USD), so the fee in token units is gas used × effective gas price ÷ 10^12, rounded up. `showFee` also checks that the receipt holds a TIP-20 `Transfer` of exactly that amount to the FeeManager, and it asks the FeeManager which token the block's proposer wants. The proposer is `block.miner`, the address that collects the block's fees.
* **Step 1, which token pays.** Three receipts, three fee tokens. The plain alphaUSD transfer pays in alphaUSD, because the transaction is a TIP-20 `transfer` and nothing else was chosen. With `feeToken: betaUSD`, the same transfer pays in betaUSD; viem sends a Tempo Transaction (type `0x76`) to carry that field. The `approve` is not a transfer, so it falls back to pathUSD. The first transfer costs more than the second (0.000319 vs 0.000024) because it creates new state: the payer's first nonce and the merchant's first alphaUSD balance each add about 250,000 gas under TIP-1000. The fees are tiny because the testnet base fee was 600,000,000 attodollars per gas, the floor in Tempo's gas parameters.
* **Step 2, the Fee AMM behind it.** The FeeManager returns each proposer's chosen token, or pathUSD if it never chose. The 20-block sample shows the mix at that moment; the betaUSD proposer shows up in some runs and not in others. The four numbers come from the contract itself: `M = 9970` over `SCALE = 10000` means a fee swap pays out 0.997 per unit paid, `N = 9985` is the price for rebalancing, and `MIN_LIQUIDITY = 1000` is the number of LP units locked on a pool's first deposit. Then, for each step 1 fee whose token differs from what its proposer wanted, the script prints that pool's reserves with `amm.getPool`. Pools are one-way: `alphaUSD -> pathUSD` turns alphaUSD fees into pathUSD and is a different pool from `pathUSD -> alphaUSD`. If a step 1 transaction lands with the betaUSD proposer, you get that pair instead, for example `alphaUSD -> betaUSD`, and a betaUSD fee paid to the betaUSD proposer needs no swap, so no pool is printed for it. Everyone on testnet shares these faucet-token pools and other people's fees move them in every block, so the script shows their reserves but measures the rate on a pool of its own in step 3.
* **Step 3, the refusal.** The TIP-20 factory creates a new USD stablecoin with a unique symbol per run (`D10…`). Its quote token defaults to pathUSD. The payer grants itself the issuer role and mints 1,000. Sending 10 of it with nothing set makes the new coin the fee token, and the node refuses the transaction before it enters the mempool (the node's waiting room for transactions). The node checks that the fee can be converted for at least one token wanted by the proposers of the last 10 blocks, directly or through the coin's quote token, and every pool for this coin is empty. `required: 419` is the worst-case fee it checked, 0.000419, not the 0.000169 the payment really costs. The balance is still 1,000 because nothing reached a block.
* **Step 3, the fix.** One `amm.mint` call seeds the `D10… -> pathUSD` pool. `userTokenAddress` is the coin users pay with, `validatorTokenAddress` is the token proposers are paid in, and you deposit only the validator side. No `approve` is needed, because since T5 Tempo's precompiles can pull TIP-20 tokens without one. On a pool's first deposit, the LP supply is half the deposit (50,000,000 units for 100 pathUSD), and 1,000 of those units are locked forever; the docs say this protects the pool's reserves from attacks. The same payment now goes through, and its receipt says the new coin paid. The script reads the pool at the block before the payment and at the payment's block: 169 units of the new coin in, 168 pathUSD units out, exactly floor(169 × 9970 / 10000). Nobody else knows this coin, so nothing else can have moved this pool.
* **Two hops.** In some runs the payment lands in a block from the betaUSD proposer. One of our test runs printed `proposer wants betaUSD` and `pool in that block: +169 D10MUZYW8E6 units in, -168 pathUSD units out (then pathUSD -> betaUSD in a shared pool)`, and still passed the check. There is no `D10… -> betaUSD` pool, so the FeeManager uses the two-hop route added in T5: new coin → pathUSD in our pool (the same +169 / −168), then pathUSD → betaUSD in a shared pool, where the proposer gets floor(168 × 0.997) = 167 betaUSD units. In that block, six other users' pathUSD fees went through the same shared pool, which moved by 6,547 / 6,523 units in total, so the second hop cannot be told apart from theirs. That is why the demo measures only its own pool.
* **Step 4, the account default.** `setUserToken` stores a default fee token for the payer in the FeeManager. Its own fee is already paid in the new coin: viem sends it as a plain EIP-1559 transaction, and for a non-Tempo transaction that calls `setUserToken`, the protocol uses the token being set. After that, the alphaUSD transfer that paid in alphaUSD in step 1 pays in the new coin, because the account level ranks above the token being moved, and so does the `approve` that fell back to pathUSD. The `approve` is cheaper than in step 1 because the allowance already exists. All four fees went through the new pool, which now holds 0.000376 of the new coin (169 + 167 + 24 + 16) and has paid out 372 pathUSD units. A transaction-level `feeToken` would still beat this default.

## 4. Student Exercise

In step 3, change `validatorTokenAmount: parseUnits("100", 6)` to `validatorTokenAmount: parseUnits("0.002", 6)`. Predict what happens, then run the script again. Steps 1 and 2, the refusal and the unchanged balance print as before. Then the script stops at the seed with `The contract function "mint" reverted`, `Error: InsufficientLiquidity()` and `Details: execution reverted: TIP fee AMM error: InsufficientLiquidity(InsufficientLiquidity)`. viem asks the node to estimate the call before signing it, and the call reverts during that estimate, so nothing is sent and no fee is paid. Why: on a pool's first deposit, the FeeManager creates LP units equal to half the deposit and must lock 1,000 of them. Half of 2,000 micro-units is 1,000, which leaves nothing for the depositor.

Now change it to `parseUnits("0.002002", 6)` and run again. The run completes. The seed line shows `LP units: 1 for the payer + 1000 locked forever`, every fee payment in steps 3 and 4 still succeeds, and the last line shows `0.00163 pathUSD to pay out`. A pool this small works because each check only needs one transaction's worst-case fee to fit in the pool, and that was about 0.0004 at the testnet base fee on October 8, 2026. A transaction with a much larger gas limit would be refused.

## Production Takeaway

If you issue a stablecoin, seeding its Fee AMM pool is part of the launch. Deposit pathUSD, or a token validators actually want, with `amm.mint`. Size the pool for the worst-case fee of your users' transactions, not their average fee, and watch it with `amm.getPool` so you can add liquidity or `rebalanceSwap` before the validator side runs dry. Check which tokens proposers want by reading `fee.getValidatorToken` for recent blocks, instead of trusting a list in the docs. In apps, set the fee token explicitly (per transaction, or once with `setUserToken`) so users are never surprised by which token paid, and read it back from `receipt.feeToken`. Treat `insufficient liquidity in FeeAMM pool` as "this token cannot pay fees right now": nothing was charged, so offer another fee token and send again.

## References

* [Fee AMM overview: converting stablecoin fees](https://tempo.xyz/developers/docs/protocol/fees/fee-amm)
* [Fee AMM specification](https://tempo.xyz/developers/docs/protocol/fees/spec-fee-amm)
* [Fee specification: fee token preferences and validator preferences](https://tempo.xyz/developers/docs/protocol/fees/spec-fee)
* [Exchanging stablecoins: exchange vs. Fee AMM](https://tempo.xyz/developers/docs/protocol/exchange)
* [Pay Fees in Any Stablecoin](https://tempo.xyz/developers/docs/guide/payments/pay-fees-in-any-stablecoin)
* [Use Your Stablecoin for Fees](https://tempo.xyz/developers/docs/guide/issuance/use-for-fees)
* [Managing Fee Liquidity](https://tempo.xyz/developers/docs/guide/stablecoin-dex/managing-fee-liquidity)
* [T5 Network Upgrade (two-hop FeeAMM routing, implicit approvals)](https://tempo.xyz/developers/docs/protocol/upgrades/t5)
* [TIP-1033: Two-Hop FeeAMM Routing](https://tips.sh/1033)
* [TIP-1000: State Creation Cost Increase](https://tips.sh/1000)
* [T12 Network Upgrade](https://tempo.xyz/developers/docs/protocol/upgrades/t12)
* Tempo node source at v1.16.0, the T12 release: [Fee AMM precompile](https://github.com/tempoxyz/tempo/blob/v1.16.0/crates/precompiles/src/tip_fee_manager/amm.rs), [FeeManager](https://github.com/tempoxyz/tempo/blob/v1.16.0/crates/precompiles/src/tip_fee_manager/mod.rs), [mempool liquidity check](https://github.com/tempoxyz/tempo/blob/v1.16.0/crates/transaction-pool/src/amm.rs) and [the refusal message](https://github.com/tempoxyz/tempo/blob/v1.16.0/crates/revm/src/error.rs)
* [viem: amm.mint](https://viem.sh/tempo/actions/amm.mint), [amm.getPool](https://viem.sh/tempo/actions/amm.getPool), [amm.burn](https://viem.sh/tempo/actions/amm.burn) and [fee.setUserToken](https://viem.sh/tempo/actions/fee.setUserToken)
