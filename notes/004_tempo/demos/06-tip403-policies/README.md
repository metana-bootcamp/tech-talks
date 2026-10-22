# Demo 06: TIP-403 Policies

On Tempo testnet, an issuer launches its own USD stablecoin (DUSD) and locks it to a TIP-403 whitelist, then a receiver (Treasury) adds its own receive policy on top. Students watch transfers get credited, revert, or succeed while the money lands in `ReceivePolicyGuard` instead of Treasury's balance, and then watch a recovery account release the held funds. The lesson: the token policy decides who may use a token at all and fails by reverting; the receive policy decides what a recipient will accept and fails by holding the funds for recovery.

## Questions This Demo Answers

* Is a token policy the same as a receive policy? No: the token policy is chosen by the token's admin and checks the sender and recipient of every transfer of that token, while the receive policy is chosen by the receiving address and filters which senders and tokens it accepts. Both are ordinary whitelist/blacklist policies stored in the same TIP-403 registry at `0x403c000000000000000000000000000000000000`.
* Which check runs first? The token policy runs first and has the final say: when it fails, the call reverts with `PolicyForbids` before anything is mined ([b], [i]), and only calls that pass it reach the receive policy ([d], [e]). In [i] both layers reject Bob and the result is still a revert, with nothing parked in the guard.
* Does a blocked transfer always disappear? No. A receive-policy block still succeeds on-chain: the funds go to `ReceivePolicyGuard` (`0xB10C000000000000000000000000000000000000`), a `TransferBlocked` event records a receipt, and the recovery authority can later release them to the receiver ([g], a resume) or somewhere else ([h], a reroute).
* Who can change policy? The token admin (the factory gives the creator `DEFAULT_ADMIN_ROLE`) picks which policy the token uses and the policy's admin edits its member list; anyone else gets `Unauthorized`. A receive policy can only be set by the address it protects (`setReceivePolicy` always configures the caller), and held funds can be claimed only by the recovery authority recorded in the receipt, which is why Bob's own claim fails with `UnauthorizedClaimer` ([f]).
* Does the issuer have to be on its own whitelist? Not to mint: a mint checks only the recipient, so the Issuer (not on the list) mints to Alice and Bob while its mint to Outsider reverts with `PolicyForbids`. Like any holder, it would need to be on the list to send or receive DUSD.
* Why test blocked flows? Because the two failures look different to your code: a revert throws, but a held transfer returns a successful receipt whose `Transfer` event points at the guard, so code that only checks `status === "success"` would mark it as paid. Preflight with `validateReceivePolicy` (the "registry says" line) and index `TransferBlocked` to catch it.

## 1. Create The Project

```bash
mkdir tempo-demo-06-tip403-policies
cd tempo-demo-06-tip403-policies
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/tip403Policies.ts`:

```ts
import { formatUnits, parseUnits, type Address, type Hex } from "viem";
import { generatePrivateKey } from "viem/accounts";
import { Account, Addresses, ExecutionError, ReceivePolicyReceipt, createClient, http } from "viem/tempo";

const alphaUsd: Address = "0x20c0000000000000000000000000000000000001"; // faucet stablecoin
const usd = (amount: bigint) => formatUnits(amount, 6); // every token here has 6 decimals
const txLink = (hash: Hex) => `https://explore.testnet.tempo.xyz/tx/${hash}`;

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

// A read-only client, plus one throwaway wallet per role. Wallets pay fees in pathUSD:
// by default Tempo charges a transfer's fee in the token being moved, and a brand-new
// token has no Fee AMM liquidity to pay fees with.
const reader = createClient({ testnet: true, transport });
const newWallet = () =>
  createClient({
    account: Account.fromSecp256k1(generatePrivateKey()),
    testnet: true,
    feeToken: Addresses.pathUsd,
    transport,
  });
type Wallet = ReturnType<typeof newWallet>;

const balanceOf = async (token: Address, account: Address) =>
  (await reader.token.getBalance({ token, account })).amount;

// Expected failures become one line, e.g. "PolicyForbids: Forbidden by policy."
function decodeRevert(error: unknown) {
  const decoded = ExecutionError.from(error as Error);
  if (decoded.errorName === "unknown") throw error; // not a protocol revert: fail loudly
  return decoded;
}

const names = new Map<string, string>(); // address -> role, for readable output
const summary: [string, string, string, string][] = [];
const surprises: string[] = [];

// Send a TIP-20 transfer and report what really happened on-chain.
async function send(
  label: string,
  from: Wallet,
  recipient: Wallet | Address,
  amount: string,
  token: Address,
  expected: string,
) {
  // viem leaves transferPolicyId undefined for pathUSD, which uses policy 1
  const { symbol, transferPolicyId = 1n } = await reader.token.getMetadata({ token });
  const sender = from.account.address;
  const to = typeof recipient === "string" ? recipient : recipient.account.address;
  const who = names.get(to) ?? to;

  // Ask the TIP-403 registry what each layer will say (the preflight a wallet can show).
  const senderOk = await reader.policy.isAuthorized({ policyId: transferPolicyId, user: sender });
  const recipientOk = await reader.policy.isAuthorized({ policyId: transferPolicyId, user: to });
  const tokenLayer = senderOk && recipientOk ? "pass" : `FAIL (${senderOk ? "recipient" : "sender"})`;
  const check = await reader.receivePolicy.validate({ token, sender, receiver: to });
  const receiveLayer = check.authorized ? "pass" : `FAIL (${check.blockedReason})`;

  console.log(`\n  ${label}: ${amount} ${symbol}`);
  console.log(`      registry says: token policy ${tokenLayer}, receive policy ${receiveLayer}`);
  const value = parseUnits(amount, 6);
  const before = await balanceOf(token, to);
  let outcome: string;
  let held: Hex | undefined;
  try {
    const { receipt } = await from.token.transferSync({ token, to, amount: value });
    const after = await balanceOf(token, to);
    [held] = ReceivePolicyReceipt.fromTransactionReceipt(receipt); // TransferBlocked log, if any
    if (held) {
      const { blockedNonce, blockedReason } = ReceivePolicyReceipt.decode(held);
      const parked = await reader.receivePolicy.getBlockedBalance({ receipt: held });
      outcome = "HELD BY GUARD";
      console.log(`      ${outcome}: tx succeeded, ${who} balance ${usd(before)} -> ${usd(after)}`);
      console.log(`      guard holds ${usd(parked)} ${symbol}, receipt #${blockedNonce} (${blockedReason})`);
    } else {
      outcome = after - before === value ? "CREDITED" : "NOT CREDITED";
      console.log(`      ${outcome}: ${who} balance ${usd(before)} -> ${usd(after)} ${symbol}`);
    }
    console.log(`      ${txLink(receipt.transactionHash)}`);
  } catch (error) {
    const { errorName, message } = decodeRevert(error);
    outcome = `REVERTED (${errorName})`;
    console.log(`      ${outcome}: ${message} Rejected before mining, nothing moved.`);
  }
  summary.push([`${label} ${amount} ${symbol}`, tokenLayer, receiveLayer, outcome]);
  if (!outcome.startsWith(expected)) surprises.push(`${label}: expected ${expected}, got ${outcome}`);
  return held;
}

// Release a held receipt from the guard to `destination`. Only the recovery authority may.
async function claim(label: string, caller: Wallet, receipt: Hex, destination: Wallet, expected: string) {
  const { token } = ReceivePolicyReceipt.decode(receipt);
  const to = destination.account.address;
  const who = names.get(to) ?? to;
  const { symbol } = await reader.token.getMetadata({ token });
  console.log(`\n  ${label}`);
  const before = await balanceOf(token, to);
  let outcome: string;
  try {
    const claimed = await caller.receivePolicy.claimSync({ to, receipt });
    const after = await balanceOf(token, to);
    outcome = "CLAIMED";
    console.log(
      `      ${outcome}: guard released ${usd(claimed.amount)} ${symbol}, ` +
        `${who} balance ${usd(before)} -> ${usd(after)}`,
    );
    console.log(`      ${txLink(claimed.receipt.transactionHash)}`);
  } catch (error) {
    const { errorName, message } = decodeRevert(error);
    outcome = `REVERTED (${errorName})`;
    console.log(`      ${outcome}: ${message}`);
  }
  if (!outcome.startsWith(expected)) surprises.push(`${label}: expected ${expected}, got ${outcome}`);
}

console.log("Demo 06: TIP-403 token policies vs receive policies (Tempo testnet, chain 42431)");

console.log("\nStep 0  Throwaway accounts, funded by the faucet (pathUSD, alphaUSD, betaUSD, thetaUSD)");
const issuer = newWallet();
const alice = newWallet();
const bob = newWallet();
const treasury = newWallet();
const treasuryOps = newWallet();
const outsider = Account.fromSecp256k1(generatePrivateKey()).address; // never funded or approved
const cast = { Issuer: issuer, Alice: alice, Bob: bob, Treasury: treasury, TreasuryOps: treasuryOps };
for (const [name, wallet] of Object.entries(cast)) {
  await wallet.faucet.fundSync({ account: wallet.account, timeout: 60_000 });
  names.set(wallet.account.address, name);
  console.log(`  ${name.padEnd(12)} ${wallet.account.address}`);
}
names.set(outsider, "Outsider");
console.log(`  ${"Outsider".padEnd(12)} ${outsider} (not funded)`);

console.log("\nStep 1  Issuer creates a USD stablecoin with the TIP-20 factory");
const created = await issuer.token.createSync({ name: "Demo USD", symbol: "DUSD", currency: "USD" });
const dusd = created.token;
console.log(`  DUSD token ${dusd}`);
console.log(`  ${txLink(created.receipt.transactionHash)}`);
await issuer.token.grantRolesSync({ token: dusd, roles: ["issuer"], to: issuer.account.address });
console.log("  The factory made the Issuer token admin; the Issuer grants itself ISSUER_ROLE to mint");
const startPolicy = (await reader.token.getMetadata({ token: dusd })).transferPolicyId;
console.log(`  DUSD starts on transfer policy ${startPolicy} (built-in always-allow)`);

console.log("\nStep 2  Issuer creates a whitelist in the shared TIP-403 registry and attaches it");
const kyc = await issuer.policy.createSync({
  type: "whitelist",
  addresses: [alice.account.address, bob.account.address, treasury.account.address],
});
console.log(`  Registry ${Addresses.tip403Registry} created policy ${kyc.policyId}`);
console.log("  Whitelist members: Alice, Bob, Treasury");
await issuer.token.changeTransferPolicySync({ token: dusd, policyId: kyc.policyId });
const nowPolicy = (await reader.token.getMetadata({ token: dusd })).transferPolicyId;
console.log(`  DUSD now enforces policy ${nowPolicy}`);
await issuer.token.mintSync({ token: dusd, to: alice.account.address, amount: parseUnits("500", 6) });
await issuer.token.mintSync({ token: dusd, to: bob.account.address, amount: parseUnits("500", 6) });
console.log("  Minted 500 DUSD each to Alice and Bob (mint checks the recipient, not the Issuer)");
try {
  await issuer.token.mintSync({ token: dusd, to: outsider, amount: parseUnits("500", 6) });
  surprises.push("Mint to Outsider: expected REVERTED, got MINTED");
} catch (error) {
  const { errorName, message } = decodeRevert(error);
  console.log(`  Mint 500 DUSD to Outsider: REVERTED (${errorName}): ${message}`);
}

console.log("\nStep 3  Token policy (issuer-controlled): may this address use DUSD at all?");
await send("[a] Alice -> Treasury", alice, treasury, "100", dusd, "CREDITED");
await send("[b] Alice -> Outsider", alice, outsider, "100", dusd, "REVERTED");

console.log("\nStep 4  Receive policy (receiver-controlled): what is Treasury willing to accept?");
const acceptedSenders = await treasury.policy.createSync({
  type: "whitelist",
  addresses: [alice.account.address],
});
const acceptedTokens = await treasury.policy.createSync({ type: "whitelist", addresses: [dusd] });
await treasury.receivePolicy.setSync({
  senderPolicyId: acceptedSenders.policyId,
  tokenPolicyId: acceptedTokens.policyId,
  claimer: treasuryOps.account.address, // recovery authority for anything held
});
const treasuryPolicy = await reader.receivePolicy.get({ account: treasury.account.address });
console.log(`  Accepted senders: policy ${treasuryPolicy.senderPolicyId} (Alice)`);
console.log(`  Accepted tokens:  policy ${treasuryPolicy.tokenPolicyId} (DUSD)`);
const authority = treasuryPolicy.recoveryAuthority;
console.log(`  Recovery authority: ${names.get(authority) ?? authority}`);
console.log(`  Refused deliveries are parked at ReceivePolicyGuard ${Addresses.receivePolicyGuard}`);
await send("[c] Alice -> Treasury", alice, treasury, "50", dusd, "CREDITED");
const heldFromBob = await send("[d] Bob -> Treasury", bob, treasury, "50", dusd, "HELD BY GUARD");
const heldAlpha = await send("[e] Alice -> Treasury", alice, treasury, "25", alphaUsd, "HELD BY GUARD");

console.log("\nStep 5  Recover held funds with the receipts from the TransferBlocked events");
if (!heldFromBob || !heldAlpha) throw new Error("Step 4 should have produced two held receipts");
await claim("[f] Bob tries to pull his held DUSD back", bob, heldFromBob, bob, "REVERTED");
await claim("[g] TreasuryOps resumes delivery to Treasury", treasuryOps, heldFromBob, treasury, "CLAIMED");
await claim("[h] TreasuryOps reroutes the alphaUSD to Alice", treasuryOps, heldAlpha, alice, "CLAIMED");

console.log("\nStep 6  Enforcement order: the token policy has the final say");
await issuer.policy.modifyWhitelistSync({
  policyId: kyc.policyId,
  address: bob.account.address,
  allowed: false,
});
console.log("  Issuer removes Bob from the DUSD whitelist (compliance freeze)");
await send("[i] Bob -> Treasury", bob, treasury, "10", dusd, "REVERTED");
console.log("  Both layers reject Bob, but nothing is held: a token-policy failure reverts the call.");

console.log("\nStep 7  Side by side: what each layer said, and what the chain did");
const header: [string, string, string, string] = ["transfer", "token policy", "receive policy", "outcome"];
for (const [transfer, tokenLayer, receiveLayer, outcome] of [header, ...summary]) {
  console.log(`  ${transfer.padEnd(37)}${tokenLayer.padEnd(18)}${receiveLayer.padEnd(25)}${outcome}`);
}

if (surprises.length > 0) {
  console.log("\nUnexpected outcomes:");
  for (const surprise of surprises) console.log(`  ${surprise}`);
  process.exitCode = 1;
} else {
  console.log("\nEvery outcome matched the lesson.");
}
```

## 3. Run The Demo

```bash
npx tsx src/tip403Policies.ts
```

Output from a real run (addresses and hashes will differ):

```text
Demo 06: TIP-403 token policies vs receive policies (Tempo testnet, chain 42431)

Step 0  Throwaway accounts, funded by the faucet (pathUSD, alphaUSD, betaUSD, thetaUSD)
  Issuer       0xBB07068aaf6958025aDccFd2aaba064A52D206AA
  Alice        0xeBC4bAeF12938A1e6F27E2464a452eea902e5B61
  Bob          0xb1E600feAC6b2072276a8050d64aFb1B04e17595
  Treasury     0xf40eA9f778D126AC824ca25Ad5F0F92B567A782d
  TreasuryOps  0x41f65562474F343f7ad39e8beD43cA9358d463D7
  Outsider     0x59ccFdb09Df69f378d224Cb29AbCE9B0f39434A7 (not funded)

Step 1  Issuer creates a USD stablecoin with the TIP-20 factory
  DUSD token 0x20C000000000000000000000A0dcb74E78865B72
  https://explore.testnet.tempo.xyz/tx/0xe172346d5f265b6211c1e6d240fae867eb071d2396e2d394adc0082a9435c3a4
  The factory made the Issuer token admin; the Issuer grants itself ISSUER_ROLE to mint
  DUSD starts on transfer policy 1 (built-in always-allow)

Step 2  Issuer creates a whitelist in the shared TIP-403 registry and attaches it
  Registry 0x403c000000000000000000000000000000000000 created policy 1252508
  Whitelist members: Alice, Bob, Treasury
  DUSD now enforces policy 1252508
  Minted 500 DUSD each to Alice and Bob (mint checks the recipient, not the Issuer)
  Mint 500 DUSD to Outsider: REVERTED (PolicyForbids): Forbidden by policy.

Step 3  Token policy (issuer-controlled): may this address use DUSD at all?

  [a] Alice -> Treasury: 100 DUSD
      registry says: token policy pass, receive policy pass
      CREDITED: Treasury balance 0 -> 100 DUSD
      https://explore.testnet.tempo.xyz/tx/0x0a1b3c7baeff0e7804d4cb818a895386f072aa2033155955d7071c72d8c31ffd

  [b] Alice -> Outsider: 100 DUSD
      registry says: token policy FAIL (recipient), receive policy pass
      REVERTED (PolicyForbids): Forbidden by policy. Rejected before mining, nothing moved.

Step 4  Receive policy (receiver-controlled): what is Treasury willing to accept?
  Accepted senders: policy 1252509 (Alice)
  Accepted tokens:  policy 1252510 (DUSD)
  Recovery authority: TreasuryOps
  Refused deliveries are parked at ReceivePolicyGuard 0xB10C000000000000000000000000000000000000

  [c] Alice -> Treasury: 50 DUSD
      registry says: token policy pass, receive policy pass
      CREDITED: Treasury balance 100 -> 150 DUSD
      https://explore.testnet.tempo.xyz/tx/0x2b07049eeeec81624e709d496d181048d28cad6dadc95ed2c1feb0e203049980

  [d] Bob -> Treasury: 50 DUSD
      registry says: token policy pass, receive policy FAIL (receivePolicy)
      HELD BY GUARD: tx succeeded, Treasury balance 150 -> 150
      guard holds 50 DUSD, receipt #2708 (receivePolicy)
      https://explore.testnet.tempo.xyz/tx/0x3f1e3540b5b15fb9de2df49c6ba647ce734882d6c03622086022f7a23baecaa3

  [e] Alice -> Treasury: 25 alphaUSD
      registry says: token policy pass, receive policy FAIL (tokenFilter)
      HELD BY GUARD: tx succeeded, Treasury balance 1000000 -> 1000000
      guard holds 25 alphaUSD, receipt #2709 (tokenFilter)
      https://explore.testnet.tempo.xyz/tx/0xfe7f1041445f42de9e4b0b2e43e50d0744c46ca3fb31089d32e0df4f4ee2025a

Step 5  Recover held funds with the receipts from the TransferBlocked events

  [f] Bob tries to pull his held DUSD back
      REVERTED (UnauthorizedClaimer): Unauthorized claimer.

  [g] TreasuryOps resumes delivery to Treasury
      CLAIMED: guard released 50 DUSD, Treasury balance 150 -> 200
      https://explore.testnet.tempo.xyz/tx/0x86b8c3e37affc8712fcaa18030d8ad562735c814320f7f294898ed112786d1f6

  [h] TreasuryOps reroutes the alphaUSD to Alice
      CLAIMED: guard released 25 alphaUSD, Alice balance 999975 -> 1000000
      https://explore.testnet.tempo.xyz/tx/0x0937ec6410a4dbc2c66843cc791c2830cf55e9170178531309257c94d2f6159d

Step 6  Enforcement order: the token policy has the final say
  Issuer removes Bob from the DUSD whitelist (compliance freeze)

  [i] Bob -> Treasury: 10 DUSD
      registry says: token policy FAIL (sender), receive policy FAIL (receivePolicy)
      REVERTED (PolicyForbids): Forbidden by policy. Rejected before mining, nothing moved.
  Both layers reject Bob, but nothing is held: a token-policy failure reverts the call.

Step 7  Side by side: what each layer said, and what the chain did
  transfer                             token policy      receive policy           outcome
  [a] Alice -> Treasury 100 DUSD       pass              pass                     CREDITED
  [b] Alice -> Outsider 100 DUSD       FAIL (recipient)  pass                     REVERTED (PolicyForbids)
  [c] Alice -> Treasury 50 DUSD        pass              pass                     CREDITED
  [d] Bob -> Treasury 50 DUSD          pass              FAIL (receivePolicy)     HELD BY GUARD
  [e] Alice -> Treasury 25 alphaUSD    pass              FAIL (tokenFilter)       HELD BY GUARD
  [i] Bob -> Treasury 10 DUSD          FAIL (sender)     FAIL (receivePolicy)     REVERTED (PolicyForbids)

Every outcome matched the lesson.
```

Walk through the output in this order:

* **Step 0:** Five fresh wallets are funded by the faucet on every run; Outsider is only an address and is never on any list. Every wallet pins pathUSD as its fee token (see the comment in the code): without that, a DUSD transfer would try to pay its fee in DUSD and fail, because a brand-new token has no Fee AMM liquidity.
* **Step 1:** The TIP-20 factory deploys DUSD at an address with the `0x20c0…` TIP-20 prefix. The creator only receives `DEFAULT_ADMIN_ROLE`, so it grants itself `ISSUER_ROLE` before minting, and every new token starts on policy 1 (always-allow).
* **Step 2:** The whitelist lives in the shared registry at `0x403c…0000`, not inside the token; the token just points at a policy ID (since the T9 upgrade the registry records that binding too), and switching it is a separate admin call. Custom IDs start at 2 in the spec, but the testnet counter is past 1,250,000, so expect a seven-digit ID. The Issuer is not on the list and still mints to Alice and Bob, because a mint checks only the recipient; that is also why the mint to Outsider reverts.
* **Step 3:** [a] is credited. [b] reverts with `PolicyForbids` and has no explorer link: the node rejected it while preparing the transaction, so no nonce was used and no fee was paid. The "registry says" line is a read made before sending (`isAuthorized` for both addresses, `validateReceivePolicy` for the receiver), and it predicted both outcomes.
* **Step 4:** Treasury builds its receive policy from two ordinary TIP-403 whitelists: one of sender addresses and one of token addresses (the token filter). [c] from Alice in DUSD is credited. [d] and [e] have explorer links, so the transactions succeeded, yet Treasury's balance did not move: the funds sit in `ReceivePolicyGuard` under a numbered receipt, for different reasons (`receivePolicy` for the sender, `tokenFilter` for the token). Open [d] in the explorer to show the `Transfer` event to `0xB10C…0000` followed by `TransferBlocked`.
* **Step 5:** A claim needs the receipt bytes from the `TransferBlocked` event, because the guard cannot list receipts. Bob sent the funds but is not the recovery authority, so [f] reverts with `UnauthorizedClaimer`. [g] claims to the original receiver, which resumes the delivery without re-checking Treasury's receive policy; [h] sends the funds anywhere else, which is a reroute and must pass the token policy and the destination's receive policy.
* **Step 6:** After the Issuer removes Bob from the DUSD whitelist, the registry says both layers reject him, and the chain reverts. Holding never rescues a transfer that breaks the token policy.
* **Step 7:** This table is the summary slide: one registry, two layers, three outcomes. Reverted means nothing moved, held means the transaction succeeded but the recipient was not credited, and credited is the only outcome where the recipient's balance went up.

## 4. Student Exercise

In Step 4, change `claimer: treasuryOps.account.address` to `claimer: "sender"` (viem's default, which stores the zero address as the recovery authority). Predict which lines change, then run it. You should see `Recovery authority: 0x0000000000000000000000000000000000000000`, [f] becomes `CLAIMED` (Bob balance 450 -> 500), and [g] and [h] revert with `UnauthorizedClaimer`. The transfers in Steps 3, 4 and 6 and the table do not change: the receive policy holds the same transfers, and only who can recover them changed. The script lists [f], [g] and [h] under "Unexpected outcomes" and exits with code 1. Update those three expected values to `"CLAIMED"`, `"REVERTED"` and `"REVERTED"`, and add `await claim("[h2] Alice takes her alphaUSD back", alice, heldAlpha, alice, "CLAIMED");` after [h]. The run passes again, and Alice gets her 25 alphaUSD back (999975 -> 1000000).

## Production Takeaway

On Tempo, a successful TIP-20 transfer no longer guarantees that the recipient was paid: track three delivery states (reverted, credited, held) and detect "held" from the `TransferBlocked` event, not from the receipt status. Preflight sends with `isAuthorized` and `validateReceivePolicy` to warn users, index `TransferBlocked` to keep the receipt bytes (the chain cannot list them, and nobody can claim without them), and choose the recovery authority on purpose, because the default lets the original sender pull the funds back.

## References

* [Tempo Policy Registry (TIP-403) Specification](https://tempo.xyz/developers/docs/protocol/tip403/spec)
* [TIP-403 receive policies](https://tempo.xyz/developers/docs/protocol/tip403/receive-policies)
* [Configure Receive Policies](https://tempo.xyz/developers/docs/guide/payments/configure-receive-policies)
* [TIP-1028: Address-Level Receive Policies](https://tips.sh/1028)
* [TIP-20 Tokens Specification](https://tempo.xyz/developers/docs/protocol/tip20/spec)
* [Managing your TIP-20 stablecoin](https://tempo.xyz/developers/docs/guide/issuance/manage-stablecoin)
* [Fee specification: fee token preferences](https://tempo.xyz/developers/docs/protocol/fees/spec-fee#fee-token-preferences)
* [T6 Network Upgrade (receive policies)](https://tempo.xyz/developers/docs/protocol/upgrades/t6)
* [viem: receivePolicy.set](https://viem.sh/tempo/actions/receivePolicy.set)
* [viem: Tempo actions](https://viem.sh/tempo)
