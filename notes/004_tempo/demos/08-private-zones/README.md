# Demo 08: Private Zones

Two throwaway users, Alice and Bob, move testnet funds into Zone A, and Alice pays Bob inside the zone. The script then asks the zone RPC what each of them is allowed to see. The lesson: a zone hides balances and payments from everyone except the accounts involved, using two layers of protection that block each other's bypasses. The operator still sees everything.

## Questions This Demo Answers

* Is a zone a private mode of the Tempo chain? No. It is a separate chain with its own chain ID and RPC. Funds enter by depositing into the zone's portal contract on Tempo and leave by withdrawing back.
* Is the deposit itself private? No. The deposit is an ordinary Tempo transaction that anyone can inspect in the explorer. Privacy covers what happens inside the zone afterwards. The current protocol also offers encrypted deposits that hide the recipient.
* How does an app get access to the zone RPC? Every request carries an `X-Authorization-Token` that the user's own key signs. The token is bound to one zone and chain ID and lasts at most 30 days. There is no API key and no gas, and requests without a valid token get HTTP 401 or 403.
* Who can read a zone balance? Only the account itself. For anyone else the token's `balanceOf` reverts, and the RPC always runs `eth_call` as the authenticated account, so you can't pretend to be someone else.
* Why are both layers needed? Each one alone has a bypass: raw storage reads (`eth_getStorageAt`) get around the contract check, and a crafted `eth_call` gets around RPC filtering. The zone RPC refuses both: raw storage reads and full blocks come back as `Sequencer only`.
* Can the operator see zone activity? Yes. The sequencer executes every zone transaction, so a zone is private from the public and from other users, not from its operator.

## 1. Create The Project

```bash
mkdir tempo-demo-08-private-zones
cd tempo-demo-08-private-zones
npm init -y
npm pkg set type=module
npm install viem@2.54.6
npm install --save-dev tsx typescript @types/node
mkdir src
```

This demo pins `viem@2.54.6` on purpose. Zone A and Zone B are an earlier testnet deployment, and Tempo's docs list 2.54.6 as the SDK version for them. Newer viem releases target the current zone protocol, which uses different chain IDs (`421700000 + zone id`).

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/zonePrivacy.ts`:

```ts
import { BaseError, type Chain, formatUnits, parseAbi, parseAbiItem, parseUnits } from "viem";
import { generatePrivateKey } from "viem/accounts";
import { Account, Actions, Storage, createClient, http } from "viem/tempo";
import { http as zoneHttp, zoneModerato } from "viem/tempo/zones";

const alphaUsd = "0x20c0000000000000000000000000000000000001";
const zoneId = 6; // Zone A, Tempo's shared testnet zone
const explorer = "https://explore.testnet.tempo.xyz/tx";

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

// A throwaway user with a Tempo testnet client and a Zone A client. zoneHttp adds the
// X-Authorization-Token header; each user gets their own token store (in a browser: sessionStorage).
function createUser() {
  const account = Account.fromSecp256k1(generatePrivateKey());
  const storage = Storage.memory();
  return {
    account,
    storage,
    tempo: createClient({ account, testnet: true, transport: http(undefined, { fetchFn: fetchWithRetry, timeout: 60_000 }) }),
    zone: createClient({
      account,
      chain: zoneModerato(zoneId),
      transport: zoneHttp(undefined, { storage, fetchFn: fetchWithRetry, timeout: 60_000 }),
    }),
  };
}

function reason(error: unknown): string {
  if (!(error instanceof BaseError)) return String(error);
  const rpcError = error.walk((cause) => typeof (cause as { code?: unknown }).code === "number");
  if (rpcError) return `${(rpcError as BaseError & { code: number }).code} ${(rpcError as BaseError).details}`;
  return error.shortMessage.replace(/ with hash "0x[0-9a-f]+"/, "").split(". ")[0];
}

async function attempt(label: string, run: () => Promise<unknown>): Promise<void> {
  try {
    console.log(`   ok     ${label}:`, await run());
  } catch (error) {
    console.log(`   error  ${label}: ${reason(error)}`);
  }
}

// Plain JSON-RPC over fetch, so the header and the HTTP status are visible
async function rawRpc(method: string, authToken?: string): Promise<string> {
  const response = await fetchWithRetry(zoneModerato(zoneId).rpcUrls.default.http[0], {
    method: "POST",
    headers: { "content-type": "application/json", ...(authToken ? { "X-Authorization-Token": authToken } : {}) },
    body: JSON.stringify({ jsonrpc: "2.0", id: 1, method, params: [] }),
  });
  if (!response.ok) return `HTTP ${response.status}`;
  return `HTTP ${response.status} ${JSON.stringify((await response.json()).result)}`;
}

const alice = createUser();
const bob = createUser();

console.log("1. Alice and Bob each deposit 25 alphaUSD from Tempo testnet into Zone A");
for (const [name, user] of [["Alice", alice], ["Bob", bob]] as const) {
  await Actions.faucet.fundSync(user.tempo, { account: user.account, timeout: 60_000 });
  const { receipt } = await Actions.zone.depositSync(user.tempo, { token: alphaUsd, amount: parseUnits("25", 6), zoneId });
  console.log(`   ${name} ${user.account.address}  deposit (public): ${explorer}/${receipt.transactionHash}`);
  // Sign the zone auth token with the user's own key: no gas, no API key
  await user.zone.zone.signAuthorizationToken({ storage: user.storage });
  while ((await user.zone.token.getBalance({ token: alphaUsd })).amount === 0n) {
    await new Promise((resolve) => setTimeout(resolve, 1_000));
  }
}

console.log("\n2. Who can talk to the Zone RPC? (raw HTTP, Alice's key)");
const now = Math.floor(Date.now() / 1000);
const scratch = Storage.memory(); // keep these test tokens out of Alice's real store
const sign = async (options: { chain?: Chain; issuedAt?: number; expiresAt?: number } = {}) =>
  (await alice.zone.zone.signAuthorizationToken({ ...options, storage: scratch })).token;
const valid = await sign();
const tampered = `${valid.slice(0, -2)}${valid.endsWith("00") ? "01" : "00"}`; // last byte of expiresAt

console.log(`   no token           ${await rawRpc("eth_chainId")}`);
console.log(`   valid token        ${await rawRpc("eth_chainId", valid)}`);
console.log(`   expired            ${await rawRpc("eth_chainId", await sign({ issuedAt: now - 7200, expiresAt: now - 3600 }))}`);
console.log(`   signed for zone 7  ${await rawRpc("eth_chainId", await sign({ chain: zoneModerato(7) }))}`);
console.log(`   31-day lifetime    ${await rawRpc("eth_chainId", await sign({ expiresAt: now + 31 * 86_400 }))}`);
console.log(`   tampered           ${await rawRpc("zone_getAuthorizationTokenInfo", tampered)}`);
console.log(`   (Alice is ${alice.account.address}: one flipped byte recovers a different, empty account)`);

console.log("\n3. Alice pays Bob 5 alphaUSD inside the zone");
const { receipt } = await alice.zone.token.transferSync({
  token: alphaUsd,
  to: bob.account.address,
  amount: parseUnits("5", 6),
  feeToken: alphaUsd,
});
console.log(`   tx ${receipt.transactionHash} (no public explorer can show this)`);

const balanceOf = parseAbi(["function balanceOf(address) view returns (uint256)"]);
const transfer = parseAbiItem("event Transfer(address indexed from, address indexed to, uint256 amount)");
const latest = await alice.zone.getBlockNumber();
const recentTransfers = (user: typeof alice, args: { from?: `0x${string}`; to?: `0x${string}` }) =>
  user.zone.getLogs({ address: alphaUsd, event: transfer, args, fromBlock: latest - 5_000n, toBlock: latest, strict: true });

console.log("\n4. What can Alice see?");
await attempt("her balance", async () => (await alice.zone.token.getBalance({ token: alphaUsd })).formatted);
await attempt("her transfers out", async () =>
  (await recentTransfers(alice, { from: alice.account.address }))
    .map((log) => `${formatUnits(log.args.amount, 6)} -> ${log.args.to}`)
    .join(", "),
);
// Execution layer: the token itself refuses to reveal other accounts
await attempt("Bob's balance", () => alice.zone.token.getBalance({ account: bob.account.address, token: alphaUsd }));
// RPC layer: eth_call always runs as the authenticated account
await attempt("Bob's balance, pretending to be Bob", () =>
  alice.zone.readContract({ account: bob.account.address, address: alphaUsd, abi: balanceOf, functionName: "balanceOf", args: [bob.account.address] }),
);
// No side doors around the contract
await attempt("raw storage slot", () => alice.zone.getStorageAt({ address: alphaUsd, slot: "0x0" }));
await attempt("latest block with transactions", () => alice.zone.getBlock({ includeTransactions: true }));

console.log("\n5. What can Bob see?");
await attempt("his balance", async () => (await bob.zone.token.getBalance({ token: alphaUsd })).formatted);
await attempt("his transfers in", async () =>
  (await recentTransfers(bob, { to: bob.account.address }))
    .map((log) => `${formatUnits(log.args.amount, 6)} <- ${log.args.from}`)
    .join(", "),
);
await attempt("Alice's receipt for that payment", () => bob.zone.getTransactionReceipt({ hash: receipt.transactionHash }));
```

## 3. Run The Demo

```bash
npx tsx src/zonePrivacy.ts
```

Output from a real run (addresses and hashes will differ):

```text
1. Alice and Bob each deposit 25 alphaUSD from Tempo testnet into Zone A
   Alice 0x1CA8F56860e5C541E4d47988c74BD2E5842B5E1A  deposit (public): https://explore.testnet.tempo.xyz/tx/0x25030c1a8845fb964554a4f32861dbc426372ac39963806b4e69017dff3c6dc1
   Bob 0x149662106F9564FC8b842cd610695cbcF18494db  deposit (public): https://explore.testnet.tempo.xyz/tx/0x455f4d7643f156c6b241bce1d726c8715d3684226b940381841ee142420ffb0a

2. Who can talk to the Zone RPC? (raw HTTP, Alice's key)
   no token           HTTP 401
   valid token        HTTP 200 "0xfb5a5046"
   expired            HTTP 403
   signed for zone 7  HTTP 403
   31-day lifetime    HTTP 403
   tampered           HTTP 200 {"account":"0x501c3d7b74f5adc454e4170f2318f11c21cb6b09","expiresAt":"0x6ac13400"}
   (Alice is 0x1CA8F56860e5C541E4d47988c74BD2E5842B5E1A: one flipped byte recovers a different, empty account)

3. Alice pays Bob 5 alphaUSD inside the zone
   tx 0x4b1d8d90e0fba1b85ea549d8fe97bcc02bfd769e858999c64963ab54d23b5728 (no public explorer can show this)

4. What can Alice see?
   ok     her balance: 19.998784
   ok     her transfers out: 5 -> 0x149662106F9564FC8b842cd610695cbcF18494db, 0.001216 -> 0xfeEC000000000000000000000000000000000000
   error  Bob's balance: -32603 execution reverted
   error  Bob's balance, pretending to be Bob: -32004 Account mismatch
   error  raw storage slot: -32005 Sequencer only
   error  latest block with transactions: -32005 Sequencer only

5. What can Bob see?
   ok     his balance: 30
   ok     his transfers in: 25 <- 0x0000000000000000000000000000000000000000, 5 <- 0x1CA8F56860e5C541E4d47988c74BD2E5842B5E1A
   error  Alice's receipt for that payment: Transaction receipt could not be found
```

The whole run takes about 10 seconds. Walk through the output in this order:

* **Step 1:** open a deposit link in the explorer. The sender, the amount and the portal are public, because a deposit is an ordinary Tempo transaction. Privacy starts once the funds are inside the zone.
* **Step 2:** the zone RPC isn't a public endpoint. A request without a token gets 401. Alice's own key signs the token, and an expired one, one for another zone, or one valid for longer than 30 days all get 403. The tampered token is not rejected: it recovers to a different address, which owns nothing. You can't become Alice without her key.
* **Step 3:** a payment inside the zone. No explorer can show it.
* **Step 4:** Alice sees her own balance and her outgoing transfers, including the 0.001216 alphaUSD fee paid to the fee manager (`0xfeEC…`). Reading Bob's balance reverts inside the token (execution layer). Pretending to be Bob is refused by the RPC (`Account mismatch`). Raw storage reads and full blocks are refused (`Sequencer only`), so both bypasses are closed.
* **Step 5:** Bob sees the money arrive, through his balance and the transfer events he is a party to. He cannot fetch Alice's transaction receipt, because receipts go only to the sender.

## 4. Student Exercise

Make Bob snoop on Alice. Append this to the script:

```ts
console.log("\nEXERCISE: Bob snoops on Alice");
await attempt("Alice's balance", () => bob.zone.token.getBalance({ account: alice.account.address, token: alphaUsd }));
await attempt("all of Alice's transfers out", async () =>
  (await recentTransfers(bob, { from: alice.account.address }))
    .map((log) => `${formatUnits(log.args.amount, 6)} -> ${log.args.to}`)
    .join(", "),
);
```

Expected: Alice's balance reverts. Bob asks for *all* of Alice's outgoing transfers but gets only the 5 alphaUSD paid to him. Her fee payment is filtered out because he isn't a party to it.

## Production Takeaway

A zone gives each account a wallet-shaped view: its own balance, its own events, and its own receipts. A global, explorer-style view is never available. Design backends around per-account tokens with short lifetimes. Never log or share a user's token. Be precise about the trust model: the public and other users can't see activity, but the operator can.

## References

* [Connect to a zone](https://tempo.xyz/developers/docs/guide/private-zones/connect-to-a-zone)
* [Zone RPC access control](https://tempo.xyz/developers/docs/protocol/zones/rpc)
* [Deposit to a zone](https://tempo.xyz/developers/docs/guide/private-zones/deposit-to-a-zone)
* [Send tokens within a zone](https://tempo.xyz/developers/docs/guide/private-zones/send-tokens-within-a-zone)
* [Tempo Zones protocol overview](https://tempo.xyz/developers/docs/protocol/zones)
