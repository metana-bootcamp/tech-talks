# Demo 02: Sponsored Payments

Students watch a user pay a merchant in alphaUSD on Tempo testnet while someone else pays the fee: first Tempo's public testnet sponsor, then a sponsor relay that the script runs itself and that co-signs only the requests its policy approves. Every outcome is read back from the chain (the receipt's `feePayer` and the fee transfer), including the requests the relay refuses and the ones the chain itself rejects. The lesson: Tempo makes the fee payer a native part of the transaction, so the product decision of who and what you sponsor is a few lines of policy code in your relay.

## Questions This Demo Answers

* Does the sponsor ever touch the user's money? No. The user's alphaUSD goes straight to the merchant (`user alphaUSD change: -10`), and the sponsor only pays the fee in its own pathUSD; its alphaUSD balance never moves (step 5).
* Can the sponsor or relay change the recipient or the amount? No. The user signs the calls (recipient and amount), nonce, gas and validity window; the sponsor signs a separate payload (prefixed with `0x78`) over the same fields plus the fee token and the user's address, so editing the amount breaks the user's signature and the chain rejects the transaction. The only thing the sponsor gets to choose is the fee token.
* What happens when a request breaks the policy: does the user pay instead? Not automatically. The relay refuses to co-sign (`Spend limit exceeded.` or `Sponsorship rejected.`), nothing reaches the chain, and the user pays only if the app sends again with `feePayer: false` (step 3e).
* What stops someone from reusing a sponsored transaction? The chain. viem sends sponsored transactions with Tempo's expiring nonces, so resubmitting the same signed bytes fails with `ExpiringNonceReplay` (step 4a), and once `validBefore` has passed the transaction is refused outright (step 4b).
* Is this an ERC-4337 paymaster? No. The fee payer signature is a field of Tempo's native transaction type, so there is no paymaster contract, bundler, or EntryPoint, and the receipt reports `feePayer` directly.
* What if the sponsor runs out of fee tokens? The relay still signs, but the chain refuses the transaction (`insufficient funds for gas * price + value`), so nothing executes and nobody is charged (step 4c). Monitor the sponsor's fee-token balance.

## 1. Create The Project

```bash
mkdir tempo-demo-02-sponsored-payments
cd tempo-demo-02-sponsored-payments
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/sponsoredPayments.ts`:

```ts
import { createServer, type Server } from "node:http";
import type { AddressInfo } from "node:net";
import {
  type Address,
  type BaseError,
  decodeFunctionData,
  formatUnits,
  getAddress,
  isAddressEqual,
  type Log,
  parseEventLogs,
  parseUnits,
} from "viem";
import { generatePrivateKey } from "viem/accounts";
import {
  Abis,
  Account,
  Actions,
  Addresses,
  createClient,
  http,
  Relay,
  TempoAddress,
  type Transaction,
  withRelay,
} from "viem/tempo";

const alphaUsd = "0x20c0000000000000000000000000000000000001";
const pathUsd = Addresses.pathUsd; // 0x20c0000000000000000000000000000000000000
const symbols: Record<string, string> = { [pathUsd]: "pathUSD", [alphaUsd]: "alphaUSD" };
const explorer = "https://explore.testnet.tempo.xyz/tx/";
const limit = parseUnits("100", 6); // the most alphaUSD our sponsor covers in one payment
const fmt = (amount: bigint) => formatUnits(amount, 6);

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
const rpc = http(undefined, { fetchFn: fetchWithRetry, timeout: 60_000 });

const chain = createClient({ testnet: true, transport: rpc });
const user = Account.fromSecp256k1(generatePrivateKey());
const sponsor = Account.fromSecp256k1(generatePrivateKey());
const merchant = Account.fromSecp256k1(generatePrivateKey()).address;
const stranger = Account.fromSecp256k1(generatePrivateKey()).address;
const names: Record<string, string> = { [user.address]: "user", [sponsor.address]: "our sponsor" };

// The sponsorship policy. Our relay runs it on every request before the sponsor signs anything:
// sponsor exactly one call, to alphaUSD, that is a payment, to the merchant, of at most 100 alphaUSD.
function policy(tx: Transaction.TransactionRequest): Relay.feePayer.Validation {
  const call = tx.calls?.length === 1 ? tx.calls[0] : undefined;
  if (!call?.to || !call.data || !isAddressEqual(TempoAddress.resolve(call.to), alphaUsd)) return false;
  const { functionName, args } = decodeFunctionData({ abi: Abis.tip20, data: call.data });
  if (functionName !== "transfer" && functionName !== "transferWithMemo") return false;
  if (!isAddressEqual(args[0], merchant)) return false;
  return args[1] <= limit ? true : "spend_limit_exceeded";
}

// Our relay: viem's built-in Relay with the fee-payer plugin, served over plain node:http.
const servers: Server[] = [];
async function startRelay(feePayer: typeof sponsor) {
  const relay = Relay.create({
    client: chain,
    plugins: [Relay.feePayer({ account: feePayer, feeToken: pathUsd, validate: policy })],
  });
  const server = createServer(async (req, res) => {
    let body = "";
    for await (const chunk of req) body += chunk;
    const headers = { "content-type": "application/json" };
    const response = await relay.fetch(new Request("http://relay", { method: "POST", headers, body }));
    res.writeHead(response.status, headers).end(await response.text());
  });
  servers.push(server);
  await new Promise<void>((resolve) => server.listen(0, "127.0.0.1", resolve));
  return `http://127.0.0.1:${(server.address() as AddressInfo).port}`;
}

// The user's client: transactions go to the chain, sponsorship requests go to the relay at `url`.
const userVia = (url: string) =>
  createClient({ account: user, testnet: true, transport: withRelay(rpc, http(url, { fetchFn: fetchWithRetry, timeout: 60_000 })) });
const pay = (client: ReturnType<typeof userVia>, to: Address, amount: string, feePayer = true) => async () =>
  (await client.token.transferSync({ token: alphaUsd, to, amount: parseUnits(amount, 6), feePayer })).receipt;
const balance = async (token: Address, account: Address) =>
  (await chain.token.getBalance({ token, account })).amount;

type Receipt = { transactionHash: `0x${string}`; feePayer?: Address; feeToken?: Address; logs: Log[] };

// Sends one transaction, then prints who paid its fee (read from the receipt) or why it was refused.
async function attempt(label: string, expected: string, send: () => Promise<Receipt>) {
  console.log(`   ${label}`);
  let outcome = "refused";
  let receipt: Receipt | undefined;
  try {
    receipt = await send();
    const payer = getAddress(receipt.feePayer!);
    const transfers = parseEventLogs({ abi: Abis.tip20, eventName: "Transfer", logs: receipt.logs });
    const fee = transfers.find((log) => isAddressEqual(log.args.to, Addresses.feeManager))!; // the fee transfer
    outcome = names[payer] ?? "Tempo's sponsor";
    console.log(`      tx: ${explorer}${receipt.transactionHash}`);
    console.log(`      fee paid by: ${outcome} ${payer} (${fmt(fee.args.amount)} ${symbols[receipt.feeToken!]})`);
  } catch (error) {
    console.log(`      refused: ${(error as BaseError).details ?? String(error)}`);
  }
  if (outcome !== expected) {
    console.log(`      UNEXPECTED: expected ${expected}`);
    process.exitCode = 1;
  }
  return receipt;
}

try {
  console.log("Demo 02: sponsored payments on Tempo testnet (Moderato, chain 42431)\n");
  console.log("1. Create throwaway accounts and fund them from the testnet faucet");
  await Actions.faucet.fundSync(chain, { account: user, timeout: 60_000 });
  await Actions.faucet.fundSync(chain, { account: sponsor, timeout: 60_000 });
  const sponsorStart = await balance(pathUsd, sponsor.address);
  console.log(`   user      ${user.address}  alphaUSD ${fmt(await balance(alphaUsd, user.address))}`);
  console.log(`   sponsor   ${sponsor.address}  pathUSD ${fmt(sponsorStart)} (pays fees for our relay)`);
  console.log(`   merchant  ${merchant}  (the only recipient our policy allows)`);

  console.log("\n2. Tempo's public testnet sponsor: the client only adds a relay URL and feePayer: true");
  const viaTempo = userVia("https://sponsor.moderato.tempo.xyz");
  let before = await balance(alphaUsd, user.address);
  await attempt("pay merchant 10 alphaUSD", "Tempo's sponsor", pay(viaTempo, merchant, "10"));
  console.log(`      user alphaUSD change: ${fmt((await balance(alphaUsd, user.address)) - before)} (the payment only)`);

  const ourUrl = await startRelay(sponsor);
  const viaOurs = userVia(ourUrl);
  console.log(`\n3. Our relay at ${ourUrl}, policy: alphaUSD payments to the merchant, max 100`);
  const approved = await attempt("a. pay merchant 25 alphaUSD", "our sponsor", pay(viaOurs, merchant, "25"));
  await attempt("b. pay merchant 250 alphaUSD (over the limit)", "refused", pay(viaOurs, merchant, "250"));
  await attempt("c. pay a stranger 5 alphaUSD (not allowlisted)", "refused", pay(viaOurs, stranger, "5"));
  await attempt("d. approve the merchant to spend 5 alphaUSD (not a payment)", "refused", async () => {
    const amount = parseUnits("5", 6);
    return (await viaOurs.token.approveSync({ token: alphaUsd, spender: merchant, amount, feePayer: true })).receipt;
  });
  before = await balance(alphaUsd, user.address);
  await attempt("e. pay merchant 250 alphaUSD, feePayer: false", "user", pay(viaOurs, merchant, "250", false));
  console.log(`      user alphaUSD change: ${fmt((await balance(alphaUsd, user.address)) - before)} (payment + fee)`);

  console.log("\n4. Rules the chain enforces by itself");
  const raw = await chain.request({ method: "eth_getRawTransactionByHash", params: [approved!.transactionHash] });
  await attempt("a. resubmit the signed bytes of 3a, copied from the chain", "refused", () =>
    chain.sendRawTransactionSync({ serializedTransaction: raw! }));
  const calls = [Actions.token.transfer.call(chain, { token: alphaUsd, to: merchant, amount: parseUnits("5", 6) })];
  const request = await viaOurs.prepareTransactionRequest({ calls, feePayer: true }); // relay co-signs
  const held = await viaOurs.signTransaction(request); // user signs; nothing is sent yet
  const validBefore = Number(request.validBefore); // viem's default window for sponsored payments
  const window = validBefore - Math.floor(Date.now() / 1000);
  console.log(`   b. sign a sponsored 5 alphaUSD payment (valid for the next ${window} s), then hold it until it expires`);
  while (Math.floor(Date.now() / 1000) <= validBefore) await new Promise((resolve) => setTimeout(resolve, 1_000));
  await attempt("   submit it after validBefore has passed", "refused", () =>
    viaOurs.sendRawTransactionSync({ serializedTransaction: held }));
  const viaBroke = userVia(await startRelay(Account.fromSecp256k1(generatePrivateKey()))); // never funded
  await attempt("c. a relay with an unfunded sponsor approves 5 alphaUSD", "refused", pay(viaBroke, merchant, "5"));

  console.log("\n5. Balances read back from the chain");
  console.log(`   our sponsor spent ${fmt(sponsorStart - (await balance(pathUsd, sponsor.address)))} pathUSD on fees`);
  console.log(`   our sponsor still holds ${fmt(await balance(alphaUsd, sponsor.address))} alphaUSD (untouched)`);
  console.log(`   merchant received ${fmt(await balance(alphaUsd, merchant))} alphaUSD, all of it from the user`);
} finally {
  for (const server of servers) server.close();
}
```

## 3. Run The Demo

```bash
npx tsx src/sponsoredPayments.ts
```

Output from a real run (addresses and hashes will differ):

```text
Demo 02: sponsored payments on Tempo testnet (Moderato, chain 42431)

1. Create throwaway accounts and fund them from the testnet faucet
   user      0x06214A4F0a0dcfc953790035Ff6c3b50b9A5Bcee  alphaUSD 1000000
   sponsor   0xb7507637e9aDF552b4F58Cc759f3265BeB720838  pathUSD 1000000 (pays fees for our relay)
   merchant  0x601F6c2EEba6dEae79CC5f92f898D6d03D4d97aB  (the only recipient our policy allows)

2. Tempo's public testnet sponsor: the client only adds a relay URL and feePayer: true
   pay merchant 10 alphaUSD
      tx: https://explore.testnet.tempo.xyz/tx/0x4a1affa594e5d877a83d5968af6052d54943c37550279a2d91cd1c7cbbb58a02
      fee paid by: Tempo's sponsor 0x58Aa7CE42E1D13B2919E2ac7E006c4fBC171442c (0.000182 pathUSD)
      user alphaUSD change: -10 (the payment only)

3. Our relay at http://127.0.0.1:35763, policy: alphaUSD payments to the merchant, max 100
   a. pay merchant 25 alphaUSD
      tx: https://explore.testnet.tempo.xyz/tx/0x48cdb99c1278cc5f9453e110976130d59cb767f1e26b77655ccf768e91b8417e
      fee paid by: our sponsor 0xb7507637e9aDF552b4F58Cc759f3265BeB720838 (0.000032 pathUSD)
   b. pay merchant 250 alphaUSD (over the limit)
      refused: Spend limit exceeded.
   c. pay a stranger 5 alphaUSD (not allowlisted)
      refused: Sponsorship rejected.
   d. approve the merchant to spend 5 alphaUSD (not a payment)
      refused: Sponsorship rejected.
   e. pay merchant 250 alphaUSD, feePayer: false
      tx: https://explore.testnet.tempo.xyz/tx/0xf181f97de9b31e08fc62bbc421e0d326b444bf1a05d7f32a2b0a02fbb2a9580e
      fee paid by: user 0x06214A4F0a0dcfc953790035Ff6c3b50b9A5Bcee (0.000169 alphaUSD)
      user alphaUSD change: -250.000169 (payment + fee)

4. Rules the chain enforces by itself
   a. resubmit the signed bytes of 3a, copied from the chain
      refused: nonce manager error: Tempo Transaction nonce error: ExpiringNonceReplay(ExpiringNonceReplay)
   b. sign a sponsored 5 alphaUSD payment (valid for the next 25 s), then hold it until it expires
      submit it after validBefore has passed
      refused: 'valid_before' 1790964842 is too close to current time (min allowed: 1790964845)
   c. a relay with an unfunded sponsor approves 5 alphaUSD
      refused: insufficient funds for gas * price + value: have 0 want 86

5. Balances read back from the chain
   our sponsor spent 0.000032 pathUSD on fees
   our sponsor still holds 1000000 alphaUSD (untouched)
   merchant received 285 alphaUSD, all of it from the user
```

Walk through the output in this order:

* **Step 1, setup.** Three fresh accounts on every run. The user and the sponsor are funded by the testnet faucet; the sponsor holds pathUSD, the token it pays fees in. The merchant is the only recipient the policy will sponsor. If the whole class shares one IP, the public RPC answers bursts with HTTP 429; `fetchWithRetry` retries those, so a run can take longer but should not fail.
* **Step 2, Tempo's public sponsor.** The only differences from a normal payment are the `withRelay(rpc, http("https://sponsor.moderato.tempo.xyz"))` transport and `feePayer: true`. Point at `fee paid by: Tempo's sponsor 0x58Aa…`, which comes from the receipt's `feePayer` field, and at `user alphaUSD change: -10`: the user paid exactly the amount and no fee. This fee (0.000182 pathUSD) is larger than in 3a because it is the merchant's first alphaUSD balance, and Tempo charges 250,000 gas to create new state.
* **Step 3a, our relay approves.** Same client code, now pointed at our own relay: viem's built-in `Relay.create` with `Relay.feePayer`, served by `node:http` on a random local port. The relay runs `policy`, approves, and the sponsor co-signs while filling the transaction (`eth_fillTransaction`). The user signs and broadcasts, and the receipt shows our sponsor paid 0.000032 pathUSD. `Relay.feePayer`'s `validate` has the same contract as `Handler.relay` from `tapimo` in the Tempo docs.
* **Steps 3b to 3d, the policy refuses.** Each refusal is a JSON-RPC error from our relay, not a chain error. Returning `"spend_limit_exceeded"` surfaces as `Spend limit exceeded.`, and returning `false` as `Sponsorship rejected.`. There is no transaction hash because nothing was sent, so nobody paid anything. Note 3d: without a policy, a user could get any call sponsored, including an `approve`.
* **Step 3e, the user pays.** With `feePayer: false` the relay does not sponsor, and the receipt names the user as fee payer. The fee is in alphaUSD because, for a self-paid TIP-20 `transfer`, the protocol uses the transferred token as the fee token. The balance change is `-250.000169`, the payment plus the fee. This fee is also higher because it is the user's first transaction with a regular nonce, which costs 250,000 gas for account creation; sponsored transactions use expiring nonces instead.
* **Step 4a, replay.** Anyone can copy a sponsored transaction's signed bytes from the chain (`eth_getRawTransactionByHash`). Resubmitting them fails with `ExpiringNonceReplay`, so the sponsor never pays twice for one signature. No policy code is involved.
* **Step 4b, expiry.** Both signatures cover `validBefore`, so a sponsor's approval is short-lived by construction: viem sets 25 seconds by default and the protocol allows at most 300. The script requests a sponsored payment without setting a window, so viem applies its default, then holds the co-signed transaction until that window has passed, and the node refuses it (`too close to current time`: it needs a few seconds of validity left). The pause of about 25 seconds is deliberate; everything else runs in a few seconds.
* **Step 4c, an empty sponsor.** A relay whose sponsor holds no pathUSD can still sign, but the chain checks that the fee payer can cover gas × max fee in the fee token. The transaction is rejected and the user is not charged.
* **Step 5, totals.** Our sponsor spent only the 3a fee and its alphaUSD is untouched. The merchant received 285 alphaUSD (10 + 25 + 250), all of it from the user.

## 4. Student Exercise

Turn the policy off. In `startRelay`, delete `validate: policy` from `Relay.feePayer({ ... })` and run the script again. Steps 3b, 3c and 3d now go through, each with `fee paid by: our sponsor` and an `UNEXPECTED: expected refused` line, so the script exits with code 1. The sponsor pays for the over-limit payment, for the payment to a stranger, and even for the `approve` call. In our run, the sponsor's fee spend in step 5 rose from 0.000032 to 0.000421 pathUSD, and the merchant received 535 alphaUSD instead of 285. Steps 4a to 4c still fail, because the chain enforces them without any policy. This is the blind sponsorship the decks warn about: without `validate`, the relay sponsors every request it receives.

## Production Takeaway

Keep the fee payer key on a server behind a relay and treat the `validate` policy as the product: allowlist targets, methods and recipients, cap amounts, rate-limit per user (record each signing commitment in `onSponsored`, keyed by `signPayload`), and authenticate callers. The protocol already enforces replay protection, expiry, and binding the sponsor's signature to the exact calls, but an unfunded sponsor turns every sponsored payment into an error, so monitor its fee-token balance. In production, use Tempo's hosted Fee Payer API, or self-host the same policy with `tapimo`'s `Handler.relay` or viem's `Relay`.

## References

* [Sponsor user fees with feePayer (Tempo docs)](https://tempo.xyz/developers/docs/guide/payments/sponsor-user-fees)
* [Tempo Fee Payer API, including the public testnet fee payer](https://tempo.xyz/developers/docs/api/fee-payer)
* [Relay & Fee Payer Handler (`tapimo`)](https://tempo.xyz/developers/docs/server/relay-handler)
* [Tempo Transactions specification: fee payer signature details](https://tempo.xyz/developers/docs/protocol/transactions/spec-tempo-transaction#fee-payer-signature-details)
* [Fee specification: fee payer and fee token preferences](https://tempo.xyz/developers/docs/protocol/fees/spec-fee)
* [Expiring nonces](https://tempo.xyz/developers/docs/guide/tempo-transaction#expiring-nonces)
* [Tempo EVM differences: state creation costs](https://tempo.xyz/developers/docs/quickstart/evm-compatibility)
* [viem: Sponsor User Fees](https://viem.sh/tempo/guides/sponsor-fees)
* [viem: Run a Relay](https://viem.sh/tempo/guides/relay/run)
* [viem: Fee Payer Plugin (`Relay.feePayer`)](https://viem.sh/tempo/relay/plugins/fee-payer)
* [viem: `withRelay`](https://viem.sh/tempo/transports/withRelay)
