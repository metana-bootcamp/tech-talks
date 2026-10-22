# Demo 05: Batch Payments

Students run a three-person payroll on Tempo testnet from a manifest: validated against live chain data, paid as ONE Tempo Transaction with one TIP-20 `transferWithMemo` call per row, and reconciled recipient by recipient. A bonus batch whose last row overdraws then shows the lesson: a Tempo batch is all or nothing, and only chain state, not a success status, proves who got paid.

## Questions This Demo Answers

* How do you pay several people in one transaction on Tempo? Put one call per payment in the `calls` list of a Tempo Transaction (type `0x76`): the three payroll rows produced one hash, one block, one fee and three `TransferWithMemo` events, with no batching contract involved.
* Where does each row's payroll reference go? Into the TIP-20 transfer memo, a fixed 32-byte field emitted (and indexed) in `TransferWithMemo(from, to, amount, memo)`. viem's `stringToHex(reference, { size: 32 })` throws `SizeOverflowError` for anything longer instead of truncating it, so check the length during validation.
* If one payment in the batch cannot succeed, does the whole batch fail? Yes. By default the node's gas estimation runs the batch and refuses it (`InsufficientBalance`, nothing broadcast, no fee); with a fixed gas limit it is mined with status `reverted`, the trace shows rows 1 and 2 executed and were rolled back, the fee is still charged, and no employee balance changes.
* Does a successful batch transaction prove every recipient got paid? Not by itself: `status: success` only means nothing reverted. Reconcile every row against the receipt's `TransferWithMemo` events and against on-chain balances, including the employer's debit.
* Why validate off-chain if the chain is atomic anyway? Validation stops bad rows (malformed address, non-positive amount, oversized memo, total above the live balance) before anything is signed or paid for. Atomicity is the backstop for whatever slips through or changes between validation and execution, and a mined revert still costs a fee.
* Which token pays the batch's fee? The `feeToken` the client sets, pathUSD here, so alphaUSD moves by exactly the manifest total. Without it, the protocol charges the fee in the token every call transfers (see the exercise).

## 1. Create The Project

```bash
mkdir tempo-demo-05-batch-payments
cd tempo-demo-05-batch-payments
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/payroll.ts`:

```ts
import { type Address, type Hash, type Hex, formatUnits, hexToString, isAddress } from "viem";
import { parseEventLogs, parseUnits, size, stringToHex, zeroAddress } from "viem";
import { generatePrivateKey } from "viem/accounts";
import { Abis, Account, createClient, ExecutionError, http } from "viem/tempo";

const pathUSD = "0x20c0000000000000000000000000000000000000";
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

// Throwaway testnet accounts. Fees are paid in pathUSD, so the payroll token
// (alphaUSD) moves by exactly the manifest total.
const employer = Account.fromSecp256k1(generatePrivateKey());
const [ana, ben, chloe] = [1, 2, 3].map(() => Account.fromSecp256k1(generatePrivateKey()).address);
const client = createClient({ account: employer, testnet: true, transport, feeToken: pathUSD });

type Row = { employee: string; to: string; amount: string; reference: string };

// The manifest is the audit object: who gets paid, how much, and the reference
// that travels on chain as the 32-byte transfer memo.
const manifest: Row[] = [
  { employee: "EMP-001", to: ana, amount: "1250.00", reference: "PAYROLL-2026-10/EMP-001" },
  { employee: "EMP-002", to: ben, amount: "980.50", reference: "PAYROLL-2026-10/EMP-002" },
  { employee: "EMP-003", to: chloe, amount: "1500.00", reference: "PAYROLL-2026-10/EMP-003" },
];

const { decimals } = await client.token.getMetadata({ token: alphaUSD });
const usd = (amount: bigint) => Number(formatUnits(amount, decimals)).toFixed(2);
const balanceOf = async (account: Address) => (await client.token.getBalance({ token: alphaUSD, account })).amount;
// Fee = ceil(gasUsed * gas price in attodollars / 10^12) units of the token in receipt.feeToken.
async function feeOf(receipt: { gasUsed: bigint; effectiveGasPrice: bigint; feeToken?: Address }) {
  const amount = (receipt.gasUsed * receipt.effectiveGasPrice + 10n ** 12n - 1n) / 10n ** 12n;
  const { symbol } = await client.token.getMetadata({ token: receipt.feeToken! });
  return `${formatUnits(amount, 6)} ${symbol}`;
}

// Off-chain checks, run against live chain data (token decimals, employer balance).
async function validate(rows: Row[]) {
  const problems: string[] = [];
  let total = 0n;
  for (const row of rows) {
    const amount = parseUnits(row.amount, decimals);
    if (!isAddress(row.to) || row.to === zeroAddress) problems.push(`${row.employee}: invalid address ${row.to}`);
    if (amount <= 0n) problems.push(`${row.employee}: amount must be positive, got ${row.amount}`);
    if (size(stringToHex(row.reference)) > 32) problems.push(`${row.employee}: reference does not fit a 32-byte memo`);
    total += amount;
  }
  const balance = await balanceOf(employer.address);
  if (total > balance) problems.push(`total ${usd(total)} exceeds the employer's balance of ${usd(balance)}`);
  return { problems, total, balance };
}

// One TIP-20 transferWithMemo call per manifest row, all in one Tempo Transaction.
const toCalls = (rows: Row[]) =>
  rows.map((row) =>
    client.token.transfer.call({
      token: alphaUSD,
      to: row.to as Address,
      amount: parseUnits(row.amount, decimals),
      memo: stringToHex(row.reference, { size: 32 }),
    }),
  );

// Decode a Tempo precompile revert (from an error or raw revert data) into one line.
// Anything that is not a decodable revert (a timeout, say) is a real problem: fail loudly.
function decode(source: unknown) {
  const error = ExecutionError.from(source as Error);
  if (error.errorName === "unknown") throw new Error(`Not a Tempo revert: ${error.message}`);
  const args = (error.args ?? []) as readonly unknown[];
  const named = error.abiItem.inputs.map(({ name }, i) => {
    const value = args[i];
    return `${name}: ${typeof value === "bigint" ? usd(value) : value}`;
  });
  return `${error.errorName}(${named.join(", ")})`;
}

console.log(`Setup: employer ${employer.address}`);
await client.faucet.fundSync({ account: employer, timeout: 60_000 });
console.log("       funded from the faucet");

console.log("\n1. Payroll manifest");
for (const row of manifest) console.log(`   ${row.employee}  ${row.to}  ${row.amount.padStart(8)}  "${row.reference}"`);

console.log("\n2. Validate the manifest against chain data");
const check = await validate(manifest);
if (check.problems.length > 0) {
  console.log(`   REJECTED, nothing submitted: ${check.problems.join("; ")}`);
  process.exit(1);
}
console.log(`   OK: total ${usd(check.total)} alphaUSD, employer holds ${usd(check.balance)} alphaUSD`);
const broken = await validate([
  { ...manifest[0], to: "0x12345" },
  { ...manifest[1], amount: "-980.50" },
  { ...manifest[2], reference: "PAYROLL-2026-10/EMP-003/RETRO-ADJUSTMENT" },
]);
console.log("   A broken copy is stopped before anything is signed:");
for (const problem of broken.problems) console.log(`     - ${problem}`);

console.log("\n3. Submit all rows as ONE Tempo Transaction");
const before = await balanceOf(employer.address);
const hash = await client.sendTransaction({ calls: toCalls(manifest) });
const receipt = await client.waitForTransactionReceipt({ hash });
console.log(`   tx     ${explorer}${receipt.transactionHash}`);
console.log(`   block  ${receipt.blockNumber}, status ${receipt.status}, fee ${await feeOf(receipt)}`);
const events = parseEventLogs({ abi: Abis.tip20, eventName: "TransferWithMemo", logs: receipt.logs });
for (const { args } of events)
  console.log(`   event  TransferWithMemo ${args.to} ${usd(args.amount)} "${hexToString(args.memo, { size: 32 })}"`);

console.log("\n4. Reconcile every row against chain state");
for (const row of manifest) {
  const onChain = await balanceOf(row.to as Address);
  const ok = onChain === parseUnits(row.amount, decimals);
  console.log(`   ${row.employee}: expected ${row.amount}, on chain ${usd(onChain)}  ${ok ? "OK" : "MISMATCH"}`);
}
const debited = before - (await balanceOf(employer.address));
const debitOk = debited === check.total ? "OK" : "MISMATCH";
console.log(`   employer debited ${formatUnits(debited, decimals)}, manifest total ${usd(check.total)}  ${debitOk}`);

console.log("\n5. Atomicity: a bonus batch whose last row overdraws the employer");
const remaining = formatUnits(await balanceOf(employer.address), decimals);
const bonus: Row[] = [
  { employee: "EMP-001", to: ana, amount: "100", reference: "BONUS-2026-10/EMP-001" },
  { employee: "EMP-002", to: ben, amount: "100", reference: "BONUS-2026-10/EMP-002" },
  { employee: "EMP-003", to: chloe, amount: remaining, reference: "BONUS-2026-10/EMP-003" },
];
console.log(`   validate() says: ${(await validate(bonus)).problems.join("; ")}`);
console.log("   We skip validation on purpose, to see what the chain itself does.");
const paidBefore = await Promise.all([ana, ben, chloe].map(balanceOf));

console.log("   a) Default send: the node estimates gas first, which runs the whole batch");
try {
  await client.sendTransaction({ calls: toCalls(bonus) });
  console.log("      UNEXPECTED: the overdrawing batch was accepted");
  process.exitCode = 1;
} catch (error) {
  console.log(`      rejected before broadcast, no fee: ${decode(error)}`);
}

console.log("   b) Fixed gas limit: nothing runs it beforehand, so it is mined and reverts");
const revertedHash = await client.sendTransaction({ calls: toCalls(bonus), gas: 1_000_000n });
const reverted = await client.waitForTransactionReceipt({ hash: revertedHash });
console.log(`      tx     ${explorer}${reverted.transactionHash}`);
console.log(`      block  ${reverted.blockNumber}, status ${reverted.status}`);
console.log(`      fee    ${await feeOf(reverted)}, charged even though nobody was paid`);
const trace = await client.request<{
  Method: "debug_traceTransaction";
  Parameters: [Hash, { tracer: "callTracer" }];
  ReturnType: { calls: { error?: string; output?: Hex }[] };
}>({ method: "debug_traceTransaction", params: [reverted.transactionHash, { tracer: "callTracer" }] });
trace.calls.forEach((call, i) => {
  const outcome = call.error ? `${call.error}: ${decode(call.output)}` : "executed, then rolled back";
  console.log(`      row ${i + 1}: ${outcome}`);
});

const paidAfter = await Promise.all([ana, ben, chloe].map(balanceOf));
const nobodyPaid = paidAfter.every((amount, i) => amount === paidBefore[i]);
console.log(`   employee balances still ${paidAfter.map(usd).join(", ")}`);
console.log(nobodyPaid ? "   Nobody in the bonus batch was paid: all or nothing." : "   MISMATCH: someone was paid");
if (!nobodyPaid) process.exitCode = 1;
```

## 3. Run The Demo

```bash
npx tsx src/payroll.ts
```

Output from a real run (addresses and hashes will differ):

```text
Setup: employer 0xD3c541A4C547F66fBb41a01EBe086758c2FC0Fb0
       funded from the faucet

1. Payroll manifest
   EMP-001  0x45a6B73CD7113dD398D565578303E28797a37F9C   1250.00  "PAYROLL-2026-10/EMP-001"
   EMP-002  0xB378fd5C11eEfd7AB7e94D16608026EDd719e5CF    980.50  "PAYROLL-2026-10/EMP-002"
   EMP-003  0x3Ab8952F19d5a4eC7Da20c7dD618242E55EA3F98   1500.00  "PAYROLL-2026-10/EMP-003"

2. Validate the manifest against chain data
   OK: total 3730.50 alphaUSD, employer holds 1000000.00 alphaUSD
   A broken copy is stopped before anything is signed:
     - EMP-001: invalid address 0x12345
     - EMP-002: amount must be positive, got -980.50
     - EMP-003: reference does not fit a 32-byte memo

3. Submit all rows as ONE Tempo Transaction
   tx     https://explore.testnet.tempo.xyz/tx/0x333a3cd70aa3461b64d8b865922b1fda4a92089fd8c86faac394f5b17e53947f
   block  37875302, status success, fee 0.000641 pathUSD
   event  TransferWithMemo 0x45a6B73CD7113dD398D565578303E28797a37F9C 1250.00 "PAYROLL-2026-10/EMP-001"
   event  TransferWithMemo 0xB378fd5C11eEfd7AB7e94D16608026EDd719e5CF 980.50 "PAYROLL-2026-10/EMP-002"
   event  TransferWithMemo 0x3Ab8952F19d5a4eC7Da20c7dD618242E55EA3F98 1500.00 "PAYROLL-2026-10/EMP-003"

4. Reconcile every row against chain state
   EMP-001: expected 1250.00, on chain 1250.00  OK
   EMP-002: expected 980.50, on chain 980.50  OK
   EMP-003: expected 1500.00, on chain 1500.00  OK
   employer debited 3730.5, manifest total 3730.50  OK

5. Atomicity: a bonus batch whose last row overdraws the employer
   validate() says: total 996469.50 exceeds the employer's balance of 996269.50
   We skip validation on purpose, to see what the chain itself does.
   a) Default send: the node estimates gas first, which runs the whole batch
      rejected before broadcast, no fee: InsufficientBalance(available: 996069.50, required: 996269.50, token: 0x20C0000000000000000000000000000000000001)
   b) Fixed gas limit: nothing runs it beforehand, so it is mined and reverts
      tx     https://explore.testnet.tempo.xyz/tx/0xbd676e68302551b59da1de24b233b7e872be4a1bffbeccb3c69f95cbca0b34f7
      block  37875307, status reverted
      fee    0.000039 pathUSD, charged even though nobody was paid
      row 1: executed, then rolled back
      row 2: executed, then rolled back
      row 3: execution reverted: InsufficientBalance(available: 996069.50, required: 996269.50, token: 0x20C0000000000000000000000000000000000001)
   employee balances still 1250.00, 980.50, 1500.00
   Nobody in the bonus batch was paid: all or nothing.
```

Walk through the output in this order:

* **Setup.** One employer funded from the faucet (1,000,000 alphaUSD and 1,000,000 pathUSD, among others) and three fresh employee addresses that have never held anything. `fetchWithRetry` only retries requests the public RPC rate-limited, and payments are broadcast with `sendTransaction` and then polled with `waitForTransactionReceipt`, because the one-call `eth_sendRawTransactionSync` returned intermittent HTTP 502s on testnet.
* **Step 1, the manifest.** This list is the audit object: who, how much, and a reference per row. The reference is what appears on chain as the memo, so finance can match every transfer to its row.
* **Step 2, validation.** Every check runs against live data: decimals come from the token's on-chain metadata and the balance from `getBalance`. The broken copy trips three checks and is never signed; the fourth check (total against balance) fires in step 5.
* **Step 3, one transaction.** One hash and one fee for three payments. The receipt decodes into three `TransferWithMemo` events whose memos round-trip to the manifest references. Every call is an allow-listed TIP-20 `transferWithMemo` and the transaction has no access list, so by the payment-lane rules (TIP-1045) it is eligible for Tempo's reserved payment blockspace; the receipt does not show the lane, so that part comes from the spec, not from this output.
* **Step 4, reconciliation.** Read balances, not just the receipt: each employee holds exactly their row, and the employer's alphaUSD went down by exactly the manifest total because the fee was paid in pathUSD.
* **Step 5a, rejected before broadcast.** `sendTransaction` first asks the node to fill in gas (`eth_fillTransaction`), which executes the batch. Row 3 needs 996,269.50 but only 996,069.50 is left after rows 1 and 2, so the node returns the TIP-20 error `InsufficientBalance`, decoded here with `ExecutionError.from` from `viem/tempo`. Nothing was broadcast and no fee was paid.
* **Step 5b, mined and reverted.** With `gas` set, nothing executes the batch beforehand: it lands in a block with status `reverted` and the fee is charged anyway. The call trace (`debug_traceTransaction`) shows rows 1 and 2 ran and were rolled back when row 3 failed, and the employee balances are unchanged. The fee is charged on gas used, not on the limit, so the generous 1,000,000 costs nothing extra; a tight limit can fail for a different reason (in a test with 400,000 gas and brand-new recipients, row 1 ran out of gas: a first transfer to a new address cost roughly 260,000 to 270,000 gas).

## 4. Student Exercise

Delete `feeToken: pathUSD` from the `createClient` call and rerun. Step 3's fee line now reads `fee 0.000635 alphaUSD`: with no fee-token preference set, the protocol charged the fee in the token every call transfers (the signed transaction carries no fee token at all). Step 4 then flags the employer: `employer debited 3730.500635, manifest total 3730.50  MISMATCH`. The payroll token paid for the fee too, which is exactly the kind of drift that row-by-row reconciliation exists to catch.

## Production Takeaway

Treat the manifest as the source of truth and the chain as the judge: validate rows against live data before signing, submit the batch as one Tempo Transaction so it settles all or nothing, and mark payroll as paid only after every row reconciles against events and balances. Pin `feeToken` so the payroll token moves by exactly the manifest total, and keep gas estimation on: it turns a doomed batch into a free, decoded error instead of a mined revert that still costs a fee.

## References

* [How to send a stablecoin payment on Tempo: batch payment transactions](https://tempo.xyz/developers/docs/guide/payments/send-a-payment)
* [Attach a Transfer Memo](https://tempo.xyz/developers/docs/guide/payments/transfer-memos)
* [TIP-20 Tokens Specification: memos, `TransferWithMemo`, `InsufficientBalance`](https://tempo.xyz/developers/docs/protocol/tip20/spec)
* [Tempo Transactions Specification: atomic `calls`](https://tempo.xyz/developers/docs/protocol/transactions/spec-tempo-transaction)
* [Fee specification: fee token preferences](https://tempo.xyz/developers/docs/protocol/fees/spec-fee)
* [Payment Lane Specification](https://tempo.xyz/developers/docs/protocol/blockspace/payment-lane-specification)
* [TIP-1045: Payment Transaction Classification](https://tips.sh/1045)
* [viem: Tempo Transactions](https://viem.sh/tempo/transactions)
* [viem: token.transfer](https://viem.sh/tempo/actions/token.transfer)
