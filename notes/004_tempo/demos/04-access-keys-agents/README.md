# Demo 04: Access Keys And Agents

A root account on Tempo testnet gives an agent its own access key that may only call `transfer()` toward one provider, spend at most 50 alphaUSD, and live for one hour; the agent then tries to overspend, pay a stranger, use another token, raise its own budget, deploy a contract, and keep paying after revocation and after expiry. Every one of those attempts is refused by the Tempo protocol, not by the script. The lesson: delegate a narrow, expiring, revocable key, never the root key.

## Questions This Demo Answers

* Is an access key just another full private key? No. It acts for the root account (step 3's payment comes from the root address), but the chain enforces its scope, budget, and expiry on every transaction, it cannot change its own permissions or deploy contracts (4d, 4e), and one root transaction revokes it (step 5).
* What happens with a token that has no limit? Once a key has any `limits`, every token without an entry has a budget of zero: `getRemainingLimit` reports 0 betaUSD and the betaUSD payment fails with `SpendingLimitExceeded` (4c). Omitting `limits` entirely does the opposite: the key reports `spendPolicy: "unlimited"` and can spend any token (we checked with a 45 alphaUSD payment from such a key).
* Can an access key deploy contracts? No. The Account Keychain spec says access-key-signed transactions cannot create contracts, including through factories, and to use the root key for deployments; in step 4e the node refuses a direct deployment with `access-key transactions cannot use CREATE as the first call`.
* Is it the SDK or the chain that enforces these rules? The chain. viem's pre-flight simulation reported 4a to 4e before anything was broadcast, but 4f skips the simulation and the transaction is mined with status `reverted` while the provider receives nothing; transactions signed by a revoked or expired key fail Tempo's transaction validation and never reach a block.
* Why does expiry matter? A forgotten or leaked agent key stays dangerous until someone remembers to revoke it; with an expiry, the session key in step 6 stops working on its own (`KeyExpired`). The node also refused to authorize keys expiring within about 10 seconds (`Access key expired: expiry … <= min allowed …`), so use realistic windows such as minutes or hours.
* Why does backend policy still matter? The protocol checks token, recipient, amount, function, and time, not intent: inside its grant, a buggy or prompt-injected agent can still pay the provider 39.99 alphaUSD for nothing. Your backend still decides whether each payment should happen (expected purchase, right price, rate limits) and keeps the root key away from the agent so it can always revoke.

## 1. Create The Project

```bash
mkdir tempo-demo-04-access-keys-agents
cd tempo-demo-04-access-keys-agents
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/agentKey.ts`:

```ts
import { formatUnits, getAddress, parseUnits } from "viem";
import type { Address, LocalAccount } from "viem";
import { generatePrivateKey, privateKeyToAddress } from "viem/accounts";
import { Account, createClient, ExecutionError, Expiry, http } from "viem/tempo";

const alphaUsd = "0x20c0000000000000000000000000000000000001";
const betaUsd = "0x20c0000000000000000000000000000000000002";
const link = (hash: string) => `https://explore.testnet.tempo.xyz/tx/${hash}`;
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

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

console.log("Demo 04: Access Keys And Agents (Tempo testnet, chain 42431)\n");

// 1. The root account owns the funds. The agent gets its own key, not the root key.
const root = Account.fromSecp256k1(generatePrivateKey());
const agent = Account.fromSecp256k1(generatePrivateKey(), { access: root });
const provider = privateKeyToAddress(generatePrivateKey()); // the API the agent is allowed to pay
const stranger = privateKeyToAddress(generatePrivateKey()); // anyone else
const client = createClient({ account: root, testnet: true, transport });
await client.faucet.fundSync({ account: root, timeout: 60_000 });
console.log("1. Root account funded from the faucet:", root.address);
console.log(`   agent key ${getAddress(agent.accessKeyAddress)} signs transactions for ${agent.address}`);

// A TIP-20 payment signed by `signer`: the root key, or an access key acting for the root account.
const pay = (signer: LocalAccount, token: Address, to: Address, amount: string) =>
  client.token.transferSync({ account: signer, token, to, amount: parseUnits(amount, 6) });

// Sends something the key should NOT allow and prints the reason Tempo gives for refusing it.
async function attempt(label: string, send: () => Promise<unknown>) {
  try {
    await send();
    console.log(`   ${label} -> UNEXPECTED: it went through`);
    process.exitCode = 1;
  } catch (error) {
    const decoded = ExecutionError.from(error as Error);
    console.log(`   ${label} -> REJECTED ${decoded.errorName === "unknown" ? decoded.message : decoded.errorName}`);
  }
}

// 2. The root key signs a key authorization: which calls, to whom, how much, until when.
const { receipt: granted } = await client.accessKey.authorizeSync({
  accessKey: agent,
  expiry: Expiry.hours(1),
  limits: [{ token: alphaUsd, limit: parseUnits("50", 6) }],
  scopes: [
    { address: alphaUsd, selector: "transfer(address,uint256)", recipients: [provider] },
    { address: betaUsd, selector: "transfer(address,uint256)", recipients: [provider] },
  ],
});
const budget = async (token: Address) =>
  formatUnits((await client.accessKey.getRemainingLimit({ accessKey: agent, token })).remaining, 6);
const key = await client.accessKey.getMetadata({ accessKey: agent });
console.log("\n2. Root authorized the agent key:", link(granted.transactionHash));
console.log(`   scope:   transfer() on alphaUSD or betaUSD, only to provider ${provider}`);
console.log(`   budget:  ${await budget(alphaUsd)} alphaUSD, ${await budget(betaUsd)} betaUSD`);
console.log(`   policy:  spendPolicy "${key.spendPolicy}", revoked: ${key.isRevoked}`);
console.log(`   expires: ${new Date(Number(key.expiry) * 1000).toISOString()}`);

// 3. Happy path: a payment inside every rule.
const paid = await pay(agent, alphaUsd, provider, "10");
console.log("\n3. Agent paid the provider 10 alphaUSD:", link(paid.receipt.transactionHash));
console.log(`   from ${getAddress(paid.receipt.from)} (the root account), signed by the agent key`);
console.log(`   budget left: ${await budget(alphaUsd)} alphaUSD (the fee was paid in alphaUSD, so it counts too)`);

// 4. Everything outside the grant is refused by Tempo, not by our code.
console.log("\n4. Agent tries what the key does not allow:");
await attempt("a. 45 alphaUSD to the provider (over budget)", () => pay(agent, alphaUsd, provider, "45"));
await attempt("b. 1 alphaUSD to a stranger", () => pay(agent, alphaUsd, stranger, "1"));
await attempt("c. 1 betaUSD to the provider (no betaUSD budget)", () => pay(agent, betaUsd, provider, "1"));
await attempt("d. raise its own budget to 1,000,000", () =>
  client.accessKey.updateLimitSync({
    account: agent, accessKey: agent, token: alphaUsd, limit: parseUnits("1000000", 6),
  }));
await attempt("e. deploy a contract", () =>
  client.deployContract({ account: agent, abi: [], bytecode: "0x600a600c600039600a6000f3602a60005260206000f3" }));
// f. Setting gas by hand skips viem's pre-flight simulation, so this one really reaches the chain.
const forced = await client.token.transfer({
  account: agent, token: alphaUsd, to: provider, amount: parseUnits("45", 6), gas: 300_000n,
});
const { status } = await client.waitForTransactionReceipt({ hash: forced });
console.log(`   f. 45 alphaUSD again, broadcast without simulation -> mined with status "${status}":`, link(forced));
if (status !== "reverted") process.exitCode = 1;
const received = await client.token.getBalance({ account: provider, token: alphaUsd });
console.log(`   provider balance: ${received.formatted} alphaUSD (only the allowed payment arrived)`);

// 5. Revocation: one root transaction locks the agent out. The root key is unaffected.
const { receipt: revoked } = await client.accessKey.revokeSync({ accessKey: agent });
console.log("\n5. Root revoked the agent key:", link(revoked.transactionHash));
await attempt("agent pays the provider 1 alphaUSD", () => pay(agent, alphaUsd, provider, "1"));
const rootPaid = await pay(root, alphaUsd, provider, "1");
console.log("   root pays the provider 1 alphaUSD -> OK", link(rootPaid.receipt.transactionHash));

// 6. Expiry: a session key that lives for 20 seconds of chain time.
const session = Account.fromSecp256k1(generatePrivateKey(), { access: root });
const expiry = Number((await client.getBlock()).timestamp) + 20;
await client.accessKey.authorizeSync({
  accessKey: session,
  expiry,
  limits: [{ token: alphaUsd, limit: parseUnits("5", 6) }],
  scopes: [{ address: alphaUsd, selector: "transfer(address,uint256)", recipients: [provider] }],
});
console.log(`\n6. Root authorized a session key that expires at chain time ${expiry} (20 s from now)`);
const early = await pay(session, alphaUsd, provider, "1");
console.log("   before expiry: session key pays 1 alphaUSD -> OK", link(early.receipt.transactionHash));
console.log("   waiting for chain time to pass the expiry...");
while ((await client.getBlock()).timestamp < BigInt(expiry)) await sleep(1000);
await attempt("after expiry: session key pays 1 alphaUSD", () => pay(session, alphaUsd, provider, "1"));

console.log("\nDone: every limit above was enforced by the Tempo protocol, not by this script.");
```

## 3. Run The Demo

```bash
npx tsx src/agentKey.ts
```

Output from a real run (addresses and hashes will differ):

```text
Demo 04: Access Keys And Agents (Tempo testnet, chain 42431)

1. Root account funded from the faucet: 0x9E7FE457B3bDDBad83e2573f7D3Cc395FAc346fe
   agent key 0x0aA66119a35b45D0fde2afc92A123ea06533029e signs transactions for 0x9E7FE457B3bDDBad83e2573f7D3Cc395FAc346fe

2. Root authorized the agent key: https://explore.testnet.tempo.xyz/tx/0xb634985cbb0930b001bdfe5f1f78bad22614857a76d37eba51f1705bd00a9bca
   scope:   transfer() on alphaUSD or betaUSD, only to provider 0x1861C7F088b8979751F5183E20e5f9393406e573
   budget:  50 alphaUSD, 0 betaUSD
   policy:  spendPolicy "limited", revoked: false
   expires: 2026-10-02T19:11:04.000Z

3. Agent paid the provider 10 alphaUSD: https://explore.testnet.tempo.xyz/tx/0xc2f512ee2c84f527d8e4f84039570224d4b48645c1d3f6fc2de6e4a71a4ab63c
   from 0x9E7FE457B3bDDBad83e2573f7D3Cc395FAc346fe (the root account), signed by the agent key
   budget left: 39.99982 alphaUSD (the fee was paid in alphaUSD, so it counts too)

4. Agent tries what the key does not allow:
   a. 45 alphaUSD to the provider (over budget) -> REJECTED SpendingLimitExceeded
   b. 1 alphaUSD to a stranger -> REJECTED CallNotAllowed
   c. 1 betaUSD to the provider (no betaUSD budget) -> REJECTED SpendingLimitExceeded
   d. raise its own budget to 1,000,000 -> REJECTED CallNotAllowed
   e. deploy a contract -> REJECTED Revm error: access-key transactions cannot use CREATE as the first call
   f. 45 alphaUSD again, broadcast without simulation -> mined with status "reverted": https://explore.testnet.tempo.xyz/tx/0xda2ab028827c439eb16f5fa429ed642df808c87065e91e9a7124ff6f75e19322
   provider balance: 10 alphaUSD (only the allowed payment arrived)

5. Root revoked the agent key: https://explore.testnet.tempo.xyz/tx/0xaf98291459bcab1cb5c64f1c134a6e84521a73d2e70b31b5bf6aec92d524ee09
   agent pays the provider 1 alphaUSD -> REJECTED Revm error: keychain validation failed: AccountKeychainError(KeyAlreadyRevoked(KeyAlreadyRevoked))
   root pays the provider 1 alphaUSD -> OK https://explore.testnet.tempo.xyz/tx/0xe889bb90b1af513f60fd8cc0280a19ab34294bb2b811f7dc95b5ea527e17fc71

6. Root authorized a session key that expires at chain time 1790964692 (20 s from now)
   before expiry: session key pays 1 alphaUSD -> OK https://explore.testnet.tempo.xyz/tx/0xa7f3ca763920ee04ce7eb96f6ffcea2453b82dab1071310e4243c70c9411fb37
   waiting for chain time to pass the expiry...
   after expiry: session key pays 1 alphaUSD -> REJECTED Revm error: keychain validation failed: AccountKeychainError(KeyExpired(KeyExpired))

Done: every limit above was enforced by the Tempo protocol, not by this script.
```

Walk through the output in this order:

* **Step 1, two keys, one account.** `Account.fromSecp256k1(key, { access: root })` builds an access-key account: it signs with its own key (the "agent key" address), but `agent.address` is the root address, so everything it sends is a root-account transaction. `fetchWithRetry` at the top only retries requests the public RPC rate-limited, so every rejection below comes from the chain.
* **Step 2, the grant.** The root signs a key authorization and sends it with `accessKey.authorizeSync`. Point at the three knobs: `scopes` (which contract, which function, which recipients), `limits` (a budget per token), and `expiry`. The budget and policy lines are read back from the chain's Account Keychain precompile: betaUSD is callable under the scope but has a budget of 0.
* **Step 3, the happy path.** A normal TIP-20 transfer that the provider sees coming from the root address. The budget drops by 10 plus the fee: with no fee-token preference set, a TIP-20 `transfer` pays its fee in that same token, and fees paid by an access key count against its budget. In a separate check, the same payment with `feeToken` set to pathUSD (no pathUSD budget) was rejected.
* **Step 4, the refusals.** a to e are refused by the node while viem simulates the transaction, with protocol errors: `SpendingLimitExceeded` (45 is more than the 39.99 left), `CallNotAllowed` (the stranger is not in the recipient allowlist), `SpendingLimitExceeded` again (no betaUSD budget), `CallNotAllowed` (the keychain precompile is not in this key's scope), and the contract-creation ban. f is the proof that the chain itself enforces the budget: the transaction is mined and reverts, and the provider's balance is still 10.
* **Step 5, revocation.** One root transaction, and the agent's next payment fails validation with `KeyAlreadyRevoked`, while the root pays normally. Per the spec, a revoked key id can never be re-authorized for that account.
* **Step 6, expiry.** A second key with a 20-second life pays once, then the script waits for chain time (block timestamps, not the laptop clock) to pass the expiry, and the identical payment fails with `KeyExpired`.

## 4. Student Exercise

Delete the `scopes` array from the agent's `authorizeSync` call in step 2 and run the demo again. 4b now goes through (`UNEXPECTED: it went through`, the stranger receives 1 alphaUSD) and the script exits with code 1; 4a and 4c are still refused with `SpendingLimitExceeded`; 4d changes from `CallNotAllowed` to `UnauthorizedCaller`, because even an unscoped access key may not manage keys; 4e is still refused. (The `scope:` line is a printed label and does not change.) Limits cap how much, scopes cap where and what, and the key hierarchy stops a key from rewriting its own rules.

## Production Takeaway

Give every agent, device, and app its own access key with the narrowest scope, a per-token budget that also covers fees, and a short expiry; keep the root key out of the agent's reach and revoke on any doubt. The protocol is the hard ceiling on what a compromised agent can do, and your backend still decides whether each payment should happen at all.

## References

* [Account Keychain Precompile](https://tempo.xyz/developers/docs/protocol/transactions/AccountKeychain)
* [TIP-1011: Enhanced Access Key Permissions](https://tempo.xyz/developers/docs/protocol/tips/tip-1011)
* [Tempo Transactions Specification: Access Keys](https://tempo.xyz/developers/docs/protocol/transactions/spec-tempo-transaction#access-keys)
* [Using Tempo Transactions: integration guides](https://tempo.xyz/developers/docs/guide/tempo-transaction/)
* [T3 Network Upgrade](https://tempo.xyz/developers/docs/protocol/upgrades/t3)
* [Fee specification: fee token preferences](https://tempo.xyz/developers/docs/protocol/fees/spec-fee)
* [viem: accessKey.authorize](https://viem.sh/tempo/actions/accessKey.authorize)
* [viem: accessKey.getRemainingLimit](https://viem.sh/tempo/actions/accessKey.getRemainingLimit)
* [viem: accessKey.revoke](https://viem.sh/tempo/actions/accessKey.revoke)
* [viem: Account.fromSecp256k1](https://viem.sh/tempo/accounts/account.fromSecp256k1)
