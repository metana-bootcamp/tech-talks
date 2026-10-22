# Demo 01: Transaction Controls

Students sign real alphaUSD payments on Tempo testnet and watch the node enforce Tempo Transaction controls: a time window that is open, one that has already closed, one that has not opened yet, two workers colliding on one sequential nonce, three parallel nonce lanes, and an expiring nonce whose replay is refused. The lesson: `validAfter`, `validBefore`, and `nonceKey` travel inside the signed payment and the network enforces them, so the authorization itself decides when, and how many times, it can execute.

## Questions This Demo Answers

* Why keep a payment authorization's window short, and what happens if it is submitted late? Signed bytes work like a bearer instrument (whoever holds them can broadcast them), so `validBefore` caps how long a leaked or delayed authorization stays usable. Submitted after its window, the node refuses it before it reaches a block (`'valid_before' ... is too close to current time (min allowed: ...)`, where "min allowed" was 2 to 3 seconds after the moment it was submitted), so it never executes and costs nothing.
* What happens to a payment whose `validAfter` is still in the future? The node accepts it into its pool and includes it in the first block whose timestamp reaches `validAfter` (in every run: exactly at `validAfter`). It refuses a `validAfter` more than 120 seconds ahead (`'valid_after' ... is too far in the future (max allowed: ...)`), so a scheduled payment is signed early but submitted shortly before its window.
* Why can't two workers share one account nonce? A sequential nonce admits one transaction per value: two payments signed with nonce 0 at the same moment produced one confirmation and one rejection (`replacement transaction underpriced`, or `nonce too low: next nonce 1, tx nonce 0` when the winner had already landed). Separate `nonceKey` lanes are independent counters, so three payments from the same account confirmed together, each lane's nonce moved to 1, and the sequential nonce did not move.
* What is an expiring nonce? A Tempo Transaction with `nonceKey = 2^256 - 1` (TIP-1009): instead of incrementing a counter, the chain remembers the transaction hash until `validBefore`, which is mandatory and, since T11, at most 300 seconds ahead (the node rejected `now + 400` with `must be within 300s of block timestamp`). Before T12 (on testnet until October 8, 2026) its `nonce` had to be `0`, and the node rejected `nonce: 5` with `expiring nonce transaction must have nonce == 0 before T12`; T12 makes that nonce configurable, so otherwise identical expiring payments can coexist.
* What stops someone from replaying a signed expiring-nonce payment? The node rejects the identical bytes with `ExpiringNonceReplay`, and once `validBefore` has passed the window check rejects them anyway. Because uniqueness is per hash, viem gives each expiring transaction a random past `validAfter` (unless you set one) so two deliberately identical payments do not collide (after T12 a distinct `nonce` does that job).
* Does a scheduled payment hold up the account's other payments? On the default sequential nonce, yes: a plain payment sent right after it waited for the window to open and landed in the same block. On its own `nonceKey` lane it blocks nothing (see the exercise).

## 1. Create The Project

```bash
mkdir tempo-demo-01-transaction-controls
cd tempo-demo-01-transaction-controls
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: each script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/timeWindows.ts`:

```ts
import { BaseError, formatUnits, InvalidInputRpcError, parseUnits } from "viem";
import { generatePrivateKey } from "viem/accounts";
import { Account, createClient, http } from "viem/tempo";

const alphaUSD = "0x20c0000000000000000000000000000000000001";
const betaUSD = "0x20c0000000000000000000000000000000000002";
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

// Throwaway testnet accounts. Payments move alphaUSD; every fee is paid in betaUSD.
const payer = Account.fromSecp256k1(generatePrivateKey());
const merchant = Account.fromSecp256k1(generatePrivateKey()).address;
const client = createClient({ account: payer, testnet: true, transport, feeToken: betaUSD });

// Windows are checked against block timestamps, not this computer's clock, so read the chain's time.
const chainTime = async () => Number((await client.getBlock()).timestamp);
const clock = (seconds: number | bigint) => new Date(Number(seconds) * 1000).toISOString().slice(11, 19);
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));
// An expected failure is the node refusing the transaction (a JSON-RPC "invalid input" error).
// Anything else, such as a timeout, is a real problem: rethrow it instead of printing it as a rejection.
const reason = (error: unknown) => {
  if (error instanceof BaseError && error.walk((e) => e instanceof InvalidInputRpcError)) return error.details;
  throw error;
};

// Sign a 5 alphaUSD payment to the merchant. viem fills in gas, fees and nonce first; the time
// window is set last, in seconds of chain time from the moment of signing, so a slow RPC cannot use it up.
// Nothing is broadcast: the signed bytes are the payment authorization, and anyone holding them can submit it.
async function authorize(window: { opensIn: number; closesIn: number; nonceKey?: bigint }) {
  const request = await client.prepareTransactionRequest({
    calls: [client.token.transfer.call({ token: alphaUSD, to: merchant, amount: parseUnits("5", 6) })],
    nonceKey: window.nonceKey,
  });
  const start = await chainTime();
  const validAfter = start + window.opensIn;
  const validBefore = start + window.closesIn;
  const serializedTransaction = await client.signTransaction({ ...request, validAfter, validBefore });
  return { serializedTransaction, validAfter, validBefore };
}

console.log(`Setup: payer ${payer.address}`);
await client.faucet.fundSync({ account: payer, timeout: 60_000 });
console.log(`       funded from the faucet, merchant ${merchant}`);

console.log("\n1. Payment inside its time window");
const payment1 = await authorize({ opensIn: -10, closesIn: 30 });
const hash1 = await client.sendRawTransaction(payment1);
const receipt1 = await client.waitForTransactionReceipt({ hash: hash1 });
const block1 = await client.getBlock({ blockNumber: receipt1.blockNumber });
console.log(`   window     ${clock(payment1.validAfter)} -> ${clock(payment1.validBefore)} UTC`);
console.log(`   included   block ${receipt1.blockNumber} at ${clock(block1.timestamp)} UTC, status ${receipt1.status}`);
// Fee = ceil(gasUsed * gas price in attodollars / 10^12) units of the receipt's fee token.
const fee1 = (receipt1.gasUsed * receipt1.effectiveGasPrice + 10n ** 12n - 1n) / 10n ** 12n;
const feeToken1 = await client.token.getMetadata({ token: receipt1.feeToken! });
console.log(`   fee        ${formatUnits(fee1, 6)} ${feeToken1.symbol} (the payment itself moved alphaUSD)`);
console.log(`   tx         ${explorer}${receipt1.transactionHash}`);

console.log("\n2. Expired authorization: signed with a 10 second window, submitted after it closed");
const payment2 = await authorize({ opensIn: -10, closesIn: 10 });
console.log(`   window     valid before ${clock(payment2.validBefore)} UTC (unix ${payment2.validBefore})`);
while ((await chainTime()) <= payment2.validBefore) await sleep(1_000); // the signed payment waits too long
try {
  await client.sendRawTransaction(payment2);
  console.log("   UNEXPECTED: the node accepted an expired payment");
  process.exitCode = 1;
} catch (error) {
  console.log(`   rejected   at ${clock(await chainTime())} UTC: ${reason(error)}`);
}

console.log("\n3. Future window: validAfter 5 seconds from now");
const payment3 = await authorize({ opensIn: 5, closesIn: 60 });
const hash3 = await client.sendRawTransaction(payment3);
console.log(`   accepted   at ${clock(await chainTime())} UTC, window opens at ${clock(payment3.validAfter)} UTC`);
const receipt3 = await client.waitForTransactionReceipt({ hash: hash3 });
const block3 = await client.getBlock({ blockNumber: receipt3.blockNumber });
console.log(`   included   block ${receipt3.blockNumber} at ${clock(block3.timestamp)} UTC, status ${receipt3.status}`);
console.log(`   tx         ${explorer}${hash3}`);

const received = await client.token.getBalance({ token: alphaUSD, account: merchant });
console.log(`\nMerchant received ${received.formatted} alphaUSD: payments 1 and 3 settled, payment 2 never did.`);
```

Create `src/nonces.ts`:

```ts
import { BaseError, type Hash, InvalidInputRpcError, maxUint256, parseUnits } from "viem";
import { generatePrivateKey } from "viem/accounts";
import { Account, createClient, http } from "viem/tempo";

const alphaUSD = "0x20c0000000000000000000000000000000000001";
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

const payer = Account.fromSecp256k1(generatePrivateKey());
const merchant = Account.fromSecp256k1(generatePrivateKey()).address;
const client = createClient({ account: payer, testnet: true, transport });

const clock = (seconds: number) => new Date(seconds * 1000).toISOString().slice(11, 19);
// An expected failure is the node refusing the transaction (a JSON-RPC "invalid input" error).
// Anything else, such as a timeout, is a real problem: rethrow it instead of printing it as a rejection.
const reason = (error: unknown) => {
  if (error instanceof BaseError && error.walk((e) => e instanceof InvalidInputRpcError)) return error.details;
  throw error;
};
const pay = (amount: string) =>
  client.token.transfer.call({ token: alphaUSD, to: merchant, amount: parseUnits(amount, 6) });
const sequentialNonce = () => client.getTransactionCount({ address: payer.address });
// Poll for a receipt by hash. Every lane starts at nonce 0, which fools viem's Ethereum-style
// "same sender + same nonce = replaced" check (it can return another lane's receipt), so turn it off.
const receiptOf = (hash: Hash) => client.waitForTransactionReceipt({ hash, checkReplacement: false, timeout: 30_000 });

console.log(`Setup: payer ${payer.address}`);
await client.faucet.fundSync({ account: payer, timeout: 60_000 });
console.log(`       funded from the faucet, merchant ${merchant}`);

console.log("\n1. Two workers share the account's single sequential nonce");
// nonceKey 0n pins the classic sequential nonce. (Left unset, viem would quietly
// switch concurrent sends from one account to expiring nonces.)
const nonce = await client.getTransactionCount({ address: payer.address, blockTag: "pending" });
const workers = await Promise.allSettled(
  ["1", "2"].map(async (amount) => {
    const hash = await client.sendTransaction({ calls: [pay(amount)], nonceKey: 0n, nonce });
    return receiptOf(hash);
  }),
);
workers.forEach((result, i) => {
  if (result.status === "fulfilled")
    console.log(`   worker ${i + 1}, nonce ${nonce}: confirmed in block ${result.value.blockNumber}`);
  else console.log(`   worker ${i + 1}, nonce ${nonce}: rejected: ${reason(result.reason)}`);
});
const landed = workers.filter((result) => result.status === "fulfilled").length;
console.log(`   ${landed} of 2 landed: one sequential nonce value admits one transaction`);
if (landed !== 1) process.exitCode = 1;

console.log("\n2. Three workers, each on its own nonce lane (nonceKey 1, 2, 3)");
const start = Date.now();
const lanes = await Promise.all(
  [1n, 2n, 3n].map(async (nonceKey) => receiptOf(await client.sendTransaction({ calls: [pay("1")], nonceKey }))),
);
lanes.forEach((receipt, i) =>
  console.log(`   lane ${i + 1}: block ${receipt.blockNumber}, ${explorer}${receipt.transactionHash}`),
);
console.log(`   all three confirmed ${Date.now() - start} ms after they were sent together`);
const laneNonces = await Promise.all(
  [1n, 2n, 3n].map((nonceKey) => client.nonce.getNonce({ account: payer.address, nonceKey })),
);
console.log(`   lane nonces now ${laneNonces.join(", ")}; sequential nonce still ${await sequentialNonce()}`);

console.log("\n3. Expiring nonce: no counter at all, just a short validity window");
const validBefore = Number((await client.getBlock()).timestamp) + 20; // chain time, not the local clock
const request = await client.prepareTransactionRequest({ calls: [pay("1")], nonceKey: "expiring", validBefore });
const signed = await client.signTransaction(request);
const receipt = await receiptOf(await client.sendRawTransaction({ serializedTransaction: signed }));
const tx = await client.getTransaction({ hash: receipt.transactionHash });
const key = tx.nonceKey === maxUint256 ? "2^256 - 1 (reserved for expiring nonces)" : tx.nonceKey;
console.log(`   nonceKey   ${key}, nonce ${tx.nonce}, valid before ${clock(validBefore)} UTC`);
console.log(`   included   block ${receipt.blockNumber}, status ${receipt.status}`);
console.log(`   tx         ${explorer}${receipt.transactionHash}`);
console.log(`   sequential nonce still ${await sequentialNonce()}`);

console.log("\n4. Replay: submit the exact same signed bytes again");
try {
  await client.sendRawTransaction({ serializedTransaction: signed });
  console.log("   UNEXPECTED: the node accepted a replay");
  process.exitCode = 1;
} catch (error) {
  console.log(`   rejected: ${reason(error)}`);
}
```

## 3. Run The Demo

```bash
npx tsx src/timeWindows.ts
npx tsx src/nonces.ts
```

Output from a real run (addresses and hashes will differ):

```text
Setup: payer 0xE22a59EFd548111f0895356aDF21531838e5aABd
       funded from the faucet, merchant 0x1bD5F27480f101bD2eD01ab15f56608F570aE6Ec

1. Two workers share the account's single sequential nonce
   worker 1, nonce 0: confirmed in block 37880863
   worker 2, nonce 0: rejected: replacement transaction underpriced
   1 of 2 landed: one sequential nonce value admits one transaction

2. Three workers, each on its own nonce lane (nonceKey 1, 2, 3)
   lane 1: block 37880865, https://explore.testnet.tempo.xyz/tx/0xaf25376fff762b3d33fd32c0ff4cf57fd91474dfe8b771b655fb0083bbeb8972
   lane 2: block 37880865, https://explore.testnet.tempo.xyz/tx/0x9092f65ea84ce673d7701450184f399ac819c19b22a5d8fc64be38e7b66804e0
   lane 3: block 37880865, https://explore.testnet.tempo.xyz/tx/0x239e316ac8ae2acb6c12de2d14afc8f9e5765a41d4b6b91f3d6a8164060a2927
   all three confirmed 810 ms after they were sent together
   lane nonces now 1, 1, 1; sequential nonce still 1

3. Expiring nonce: no counter at all, just a short validity window
   nonceKey   2^256 - 1 (reserved for expiring nonces), nonce 0, valid before 18:13:40 UTC
   included   block 37880867, status success
   tx         https://explore.testnet.tempo.xyz/tx/0xb447376253381e74424f0a0cb8e5023234bcc47655b5e0a8034e9607a09d73df
   sequential nonce still 1

4. Replay: submit the exact same signed bytes again
   rejected: nonce manager error: Tempo Transaction nonce error: ExpiringNonceReplay(ExpiringNonceReplay)
```

```text
$ npx tsx src/nonces.ts
Setup: payer 0xbC413412B79deB97c24077E52a64B760345696Bc
       funded from the faucet, merchant 0xfaB86bB8fD33bBfb03bE7eb8c025330CB9130a91

1. Two workers share the account's single sequential nonce
   worker 1, nonce 0: rejected: replacement transaction underpriced
   worker 2, nonce 0: confirmed in block 37875289
   1 of 2 landed: one sequential nonce value admits one transaction

2. Three workers, each on its own nonce lane (nonceKey 1, 2, 3)
   lane 1: block 37875291, https://explore.testnet.tempo.xyz/tx/0x7ef3965a35b3ea128a277b800feb0a02f48db24faa8460b2183ae9d14da4f2a1
   lane 2: block 37875291, https://explore.testnet.tempo.xyz/tx/0x33ebdb806f60200509bb3fcbd77d03a3c89805c8758d383c0b080424e2dc64e3
   lane 3: block 37875291, https://explore.testnet.tempo.xyz/tx/0x8e9833e7e698b7769078ccce2678594416102f42787f099ada5d3bccdae49c64
   all three confirmed 1365 ms after they were sent together
   lane nonces now 1, 1, 1; sequential nonce still 1

3. Expiring nonce: no counter at all, just a short validity window
   nonceKey   2^256 - 1 (reserved for expiring nonces), nonce 0, valid before 17:17:55 UTC
   included   block 37875294, status success
   tx         https://explore.testnet.tempo.xyz/tx/0xeffe5e8d70cfac67f2b51523ddd385299afec23bdd260b446916986d973d637b
   sequential nonce still 1

4. Replay: submit the exact same signed bytes again
   rejected: nonce manager error: Tempo Transaction nonce error: ExpiringNonceReplay(ExpiringNonceReplay)
```

Walk through the output in this order:

* **Setup.** Fresh keys are generated in memory on every run, and the faucet call (`tempo_fundAddress`) gives the payer 1,000,000 each of pathUSD, alphaUSD, betaUSD and thetaUSD in about a second. Three details keep the output honest: `fetchWithRetry` retries requests the public RPC rate-limited (it allows a few dozen per second per IP), `reason()` prints a failure only when the node itself refused the transaction (a timeout would crash the run instead of posing as protocol behavior), and every payment is broadcast first and then polled for its receipt, because the one-call `eth_sendRawTransactionSync` returned HTTP 502 in testing: always for future-dated transactions, and sometimes for the losing worker in nonces step 1.
* **timeWindows step 1, open window.** The block timestamp falls inside `validAfter -> validBefore`. Then point at the fee line: the payment moved alphaUSD, but the fee came out of betaUSD because the client sets `feeToken` (the receipt's `feeToken` confirms it). Without it (and with no account-level default), the protocol charges the fee in the token being transferred; pathUSD is only the last fallback. Gas prices are in attodollars, so the fee is `ceil(gasUsed * effectiveGasPrice / 10^12)` token units; we checked that it matches the betaUSD balance change exactly.
* **timeWindows step 2, expired.** The authorization was valid when it was signed, but the "relayer" submitted it too late and the node refused it at the door: no block, no fee. The script sets each window, and waits for it to close, in chain time (`chainTime()` reads the latest block timestamp), because that is the clock the node checks; a laptop clock can drift. The final merchant balance (10, not 15) proves it never settled. Trying to build an already-expired payment fails even earlier, at gas estimation: `transaction expired: current block timestamp ... >= validBefore ...`.
* **timeWindows step 3, future window.** The node accepted the transaction immediately and included it at exactly `validAfter`. This is why the scripts never use the synchronous send: for a future-dated transaction, `eth_sendRawTransactionSync` returned HTTP 502 after about a second, even though the node had accepted the transaction and included it later.
* **nonces step 1, one sequential nonce.** Both workers signed nonce 0; the node kept one and rejected the other with `replacement transaction underpriced` (or `nonce too low: next nonce 1, tx nonce 0` if the winner was already mined). `nonceKey: 0n` pins the classic nonce on purpose: left unset, viem notices concurrent sends from one account and quietly switches them to expiring nonces.
* **nonces step 2, nonce lanes.** Three payments sent at the same time on `nonceKey` 1, 2 and 3 confirmed within about a second, in the same or adjacent blocks, without waiting for each other. `nonce.getNonce` shows each lane at 1. Order is still guaranteed inside a lane, just not across lanes. Point at `receiptOf`: every lane starts at nonce 0, and viem's default replacement check (same sender plus same nonce means "replaced") once handed lane 3 the receipt of lane 1 in our testing, so the script waits with `checkReplacement: false`.
* **nonces step 3, expiring nonce.** The transaction as stored on chain carries `nonceKey = 2^256 - 1` and `nonce = 0`, and the sequential nonce did not move: no counter was consumed.
* **nonces step 4, replay.** The exact same signed bytes are rejected with `ExpiringNonceReplay`: the protocol keeps the hash until `validBefore`. That is the replay protection that lets expiring nonces drop the counter.

## 4. Student Exercise

Make a scheduled payment block the account, then unblock it with a nonce lane. In `src/timeWindows.ts`, step 3, add these lines right after the `accepted` log line:

```ts
const next = await client.token.transfer({ token: alphaUSD, to: merchant, amount: parseUnits("1", 6) });
const nextReceipt = await client.waitForTransactionReceipt({ hash: next });
const nextBlock = await client.getBlock({ blockNumber: nextReceipt.blockNumber });
console.log(`   follow-up  block ${nextReceipt.blockNumber} at ${clock(nextBlock.timestamp)} UTC`);
```

Run it. The follow-up payment has no time window, yet it lands in the same block as the scheduled payment, at `validAfter`: both use the sequential nonce, and the scheduled payment holds the lower value. Now move the scheduled payment to its own lane with `const payment3 = await authorize({ opensIn: 5, closesIn: 60, nonceKey: 7n });` and rerun. The follow-up confirms immediately (in our run: block 37881141 at 18:16:05, while the scheduled payment still landed at its `validAfter`, block 37881147 at 18:16:09).

## Production Takeaway

Treat time windows and nonce choice as part of the payment authorization, because the network enforces them: give anything a relayer, agent or checkout submits for a user a short `validBefore`, use expiring nonces for independent one-shot payments, and put ordered or scheduled streams on their own `nonceKey` lane so one waiting payment cannot stall the account. Broadcast and then poll for receipts (with `checkReplacement: false` once you use nonce lanes), and use a dedicated RPC in production: the public testnet endpoint rate-limits bursts per IP (HTTP 429 or a JSON-RPC "rate limited" error), which is what the scripts' `fetchWithRetry` absorbs.

## References

* [Tempo Transactions Specification](https://tempo.xyz/developers/docs/protocol/transactions/spec-tempo-transaction)
* [Using Tempo Transactions: expiring nonces, 2D nonces, scheduled transactions](https://tempo.xyz/developers/docs/guide/tempo-transaction/)
* [TIP-1009: Expiring Nonces](https://tempo.xyz/developers/docs/protocol/tips/tip-1009)
* [T11 Network Upgrade (five-minute expiring-nonce window)](https://tempo.xyz/developers/docs/protocol/upgrades/t11)
* [T12 Network Upgrade (configurable nonces for expiring transactions)](https://tempo.xyz/developers/docs/protocol/upgrades/t12)
* [Fee specification: fee token preferences and fee units](https://tempo.xyz/developers/docs/protocol/fees/spec-fee)
* [viem: Tempo Transactions](https://viem.sh/tempo/transactions)
* [viem: nonce.getNonce](https://viem.sh/tempo/actions/nonce.getNonce)
