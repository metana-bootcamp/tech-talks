# Demo 03: Payment Reconciliation

A customer sends six real payments on Tempo testnet that all point at one invoice, and a small worker decides which of them actually settles it by reading each transaction's receipt and its TIP-20 `TransferWithMemo` event from the chain. Exactly one of the six counts. The lesson: a transaction hash, a webhook, or a memo is only a claim; your backend turns it into business state after checking recipient, token, amount, memo, status, and that it has not seen the payment before.

## Questions This Demo Answers

* Is a transaction hash enough to mark an invoice paid? No. All six hashes in step 3 are real testnet transactions, but one reverted, one paid a different wallet, one paid in betaUSD, one paid 20 instead of 25, and one carried a mistyped memo; only `correct` settles the invoice.
* Is the memo enough? No. The memo only links a transfer to an invoice and anyone can send any memo: step 6 finds four successful transfers carrying this invoice's memo, and only one of them is a valid payment.
* Should a webhook mark the invoice paid by itself? No. Treat it as a wake-up signal that carries a tx hash: the worker re-reads the receipt from the chain, runs every check in `reconcile()`, and only then updates the invoice (webhook wakes the worker, the worker verifies against RPC or TIDX, then you credit).
* What happens when the customer pays with the wrong token? The betaUSD transfer succeeds on-chain and the merchant now holds 25 betaUSD, but it does not settle an alphaUSD invoice, so the invoice stays open and the payment goes to refund or review. The token is the contract that emitted the event (`payment.address`), not a field inside the event.
* Who decides what to do about underpayment or overpayment? Your product does; the chain only reports amounts. This worker rejects 20 < 25 and leaves the invoice open, and it would accept an overpayment, so a real system must decide whether to accumulate partial payments, refund the excess, or allow a tolerance.
* Why track tx hashes you have already processed? Webhooks are delivered at least once and customers resubmit hashes, so the same payment can reach the worker twice; in step 4 the retried delivery is rejected as a duplicate. In deposit-style flows there is no invoice status to flip, so this record is the only thing that prevents a double credit.

## 1. Create The Project

```bash
mkdir tempo-demo-03-reconciliation
cd tempo-demo-03-reconciliation
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

Create `src/reconcile.ts`:

```ts
import { formatUnits, hexToString, isAddressEqual, parseEventLogs, parseUnits, stringToHex } from "viem";
import type { Address, Hex } from "viem";
import { generatePrivateKey, privateKeyToAddress } from "viem/accounts";
import { Abis, Account, createClient, http } from "viem/tempo";

const alphaUsd = "0x20c0000000000000000000000000000000000001";
const betaUsd = "0x20c0000000000000000000000000000000000002";
const symbols: Record<string, string> = { [alphaUsd]: "alphaUSD", [betaUsd]: "betaUSD" };
const symbol = (token: string) => symbols[token.toLowerCase()] ?? token;
const link = (hash: string) => `https://explore.testnet.tempo.xyz/tx/${hash}`;
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));
const text = (memo: Hex) => hexToString(memo).replace(/\0/g, ""); // bytes32 memo -> readable text

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

console.log("Demo 03: Payment Reconciliation (Tempo testnet, chain 42431)\n");

// 1. The merchant only needs an address. The customer needs funds.
const merchant = privateKeyToAddress(generatePrivateKey());
const otherWallet = privateKeyToAddress(generatePrivateKey()); // another wallet the customer controls
const customer = Account.fromSecp256k1(generatePrivateKey());
const client = createClient({ account: customer, testnet: true, transport });
await client.faucet.fundSync({ account: customer, timeout: 60_000 });
console.log("1. Merchant:", merchant);
console.log("   Customer:", customer.address, "(funded from the faucet)");

// 2. The merchant's database: one open invoice. Its id, as a bytes32, is the memo to pay with.
type Invoice = { id: string; token: Address; amount: bigint; status: "open" | "paid"; paidBy?: Hex };
const invoice: Invoice = { id: `INV-${Date.now()}`, token: alphaUsd, amount: parseUnits("25", 6), status: "open" };
const memo = stringToHex(invoice.id, { size: 32 }); // text first, zero-padded on the right
const invoicesByMemo = new Map<Hex, Invoice>([[memo, invoice]]);
const processed = new Set<Hex>(); // tx hashes the worker has already handled
console.log(`\n2. Invoice ${invoice.id}: 25 alphaUSD, payable to the merchant`);
console.log(`   memo (bytes32): ${memo}`);

// 3. The customer sends six real transactions. Only one of them should settle the invoice.
console.log("\n3. Customer sends payments:");
async function pay(label: string, token: Address, to: Address, amount: string, payMemo: Hex) {
  const { receipt } = await client.token.transferSync({ token, to, amount: parseUnits(amount, 6), memo: payMemo });
  const shown = `${amount.padStart(7)} ${symbol(token).padEnd(8)} memo "${text(payMemo)}"`;
  console.log(`   ${label.padEnd(16)} ${shown}`, link(receipt.transactionHash));
  return receipt.transactionHash;
}
const wrongMemo = await pay("wrong memo", alphaUsd, merchant, "25", stringToHex(invoice.id.slice(0, -1), { size: 32 }));
const underpaid = await pay("underpaid", alphaUsd, merchant, "20", memo);
const wrongToken = await pay("wrong token", betaUsd, merchant, "25", memo);
const wrongRecipient = await pay("wrong recipient", alphaUsd, otherWallet, "25", memo);
// More than the customer owns. Setting gas by hand skips viem's simulation, so it is mined and reverts.
const reverted = await client.token.transfer({
  token: alphaUsd, to: merchant, amount: parseUnits("5000000", 6), memo, gas: 300_000n,
});
const tooMuch = `5000000 alphaUSD memo "${invoice.id}" (more than the balance)`;
console.log(`   ${"reverted".padEnd(16)} ${tooMuch}`, link(reverted));
const correct = await pay("correct", alphaUsd, merchant, "25", memo);

// 4. The worker. A webhook (or a customer saying "I paid, here is the hash") only gives it a
//    tx hash. It verifies that hash against the chain before touching the invoice.
async function reconcile(hash: Hex): Promise<string> {
  if (processed.has(hash)) return "REJECT duplicate: this tx was already processed";
  const receipt = await client.waitForTransactionReceipt({ hash });
  processed.add(hash);
  if (receipt.status !== "success") return "REJECT the tx reverted, no money moved";
  const events = parseEventLogs({ abi: Abis.tip20, eventName: "TransferWithMemo", logs: receipt.logs });
  const payment = events.find((event) => isAddressEqual(event.args.to, merchant));
  if (!payment) return "REJECT no TransferWithMemo to the merchant in this tx";
  const token = payment.address; // the TIP-20 contract that emitted the event
  const { amount, memo: paidMemo } = payment.args;
  const target = invoicesByMemo.get(paidMemo);
  if (!target) return `REJECT memo "${text(paidMemo)}" matches no invoice -> unapplied cash, manual review`;
  if (target.status === "paid") return `REJECT ${target.id} is already paid -> refund the extra payment`;
  if (!isAddressEqual(token, target.token)) {
    return `REJECT paid in ${symbol(token)}, invoice is in ${symbol(target.token)} -> refund or review`;
  }
  if (amount < target.amount) {
    return `REJECT underpaid: ${formatUnits(amount, 6)} < ${formatUnits(target.amount, 6)} -> invoice stays open`;
  }
  target.status = "paid";
  target.paidBy = hash;
  return `PAID ${target.id} with ${formatUnits(amount, 6)} ${symbol(token)}`;
}

console.log("\n4. Worker verifies each tx hash against the chain (receipt + TIP-20 logs):");
const deliveries: [string, Hex][] = [
  ["wrong memo", wrongMemo],
  ["underpaid", underpaid],
  ["wrong token", wrongToken],
  ["wrong recipient", wrongRecipient],
  ["reverted", reverted],
  ["correct", correct],
  ["webhook retried", correct], // webhooks are at-least-once: the same hash can arrive again
];
for (const [label, hash] of deliveries) console.log(`   ${label.padEnd(16)} ${await reconcile(hash)}`);

console.log(`\n5. Invoice ${invoice.id} is ${invoice.status}, settled by`, link(invoice.paidBy ?? "-"));
if (invoice.status !== "paid" || invoice.paidBy !== correct) process.exitCode = 1;

// 6. Look the memo up in the Tempo indexer (TIDX): read-only SQL over HTTP, no API key.
const params = new URLSearchParams({
  chainId: "42431",
  signature: "TransferWithMemo(address indexed from, address indexed to, uint256 amount, bytes32 indexed memo)",
  sql: `SELECT tx_hash, address, "to", amount FROM TransferWithMemo WHERE memo = '${memo}' ORDER BY block_num`,
});
console.log(`\n6. Indexer lookup (TIDX SQL): every TransferWithMemo with memo "${invoice.id}"`);
let rows: string[][] = [];
let note = "the indexer has not caught up yet (it trails the chain head by a few blocks)";
for (let attempt = 1; attempt <= 5 && !rows.some(([hash]) => hash === correct); attempt++) {
  if (attempt > 1) await sleep(2000);
  const response = await fetch(`https://indexer.testnet.tempo.xyz/query?${params}`);
  if (!response.ok) {
    note = `the indexer is unavailable right now (HTTP ${response.status})`;
    break;
  }
  rows = ((await response.json()) as { rows: string[][] }).rows;
}
for (const [hash, token, to, amount] of rows) {
  const where = isAddressEqual(to as Address, merchant) ? "merchant" : "other wallet";
  const mark = hash === invoice.paidBy ? "  <- settled the invoice" : "";
  const shown = `${formatUnits(BigInt(amount), 6).padStart(3)} ${symbol(token).padEnd(8)} -> ${where.padEnd(12)}`;
  console.log(`   ${shown} ${hash}${mark}`);
}
console.log(rows.some(([hash]) => hash === correct)
  ? `   ${rows.length} transfers carry the memo; only one of them paid the invoice.`
  : `   ${note}; the RPC checks above do not depend on it.`);
```

## 3. Run The Demo

```bash
npx tsx src/reconcile.ts
```

Output from a real run (addresses and hashes will differ):

```text
Demo 03: Payment Reconciliation (Tempo testnet, chain 42431)

1. Merchant: 0xA7723a27d6eC179d3512Fa3d889b5a52eb34A6c1
   Customer: 0x637952A124bC003f1Ec00cfb647015a9eDd11452 (funded from the faucet)

2. Invoice INV-1790961289065: 25 alphaUSD, payable to the merchant
   memo (bytes32): 0x494e562d31373930393631323839303635000000000000000000000000000000

3. Customer sends payments:
   wrong memo            25 alphaUSD memo "INV-179096128906" https://explore.testnet.tempo.xyz/tx/0xc37e5e4bffd62b548ad592e376f095482d03a2c7d8cbcf34407609ebede6f67a
   underpaid             20 alphaUSD memo "INV-1790961289065" https://explore.testnet.tempo.xyz/tx/0x03a49e3d28a3096073ba60b492b1bb3d2369c792c16777785df9b9b955fdad74
   wrong token           25 betaUSD  memo "INV-1790961289065" https://explore.testnet.tempo.xyz/tx/0x41dd98eb606a05f726e40b4618773c62c59ee877165dc6931d38c061499f7d66
   wrong recipient       25 alphaUSD memo "INV-1790961289065" https://explore.testnet.tempo.xyz/tx/0x3676c8a452cda51fc06a521e826e0b3dfc81e39a5dba26998c0be10742881723
   reverted         5000000 alphaUSD memo "INV-1790961289065" (more than the balance) https://explore.testnet.tempo.xyz/tx/0xcb5409199ec6afd1e8871cd88316292cf6b62a4367bc002a913971b9c0bcf7ee
   correct               25 alphaUSD memo "INV-1790961289065" https://explore.testnet.tempo.xyz/tx/0xf026cc1d499f4a0cfb12da492796b823ea0904c41ace7c073a764808756cebaf

4. Worker verifies each tx hash against the chain (receipt + TIP-20 logs):
   wrong memo       REJECT memo "INV-179096128906" matches no invoice -> unapplied cash, manual review
   underpaid        REJECT underpaid: 20 < 25 -> invoice stays open
   wrong token      REJECT paid in betaUSD, invoice is in alphaUSD -> refund or review
   wrong recipient  REJECT no TransferWithMemo to the merchant in this tx
   reverted         REJECT the tx reverted, no money moved
   correct          PAID INV-1790961289065 with 25 alphaUSD
   webhook retried  REJECT duplicate: this tx was already processed

5. Invoice INV-1790961289065 is paid, settled by https://explore.testnet.tempo.xyz/tx/0xf026cc1d499f4a0cfb12da492796b823ea0904c41ace7c073a764808756cebaf

6. Indexer lookup (TIDX SQL): every TransferWithMemo with memo "INV-1790961289065"
    20 alphaUSD -> merchant     0x03a49e3d28a3096073ba60b492b1bb3d2369c792c16777785df9b9b955fdad74
    25 betaUSD  -> merchant     0x41dd98eb606a05f726e40b4618773c62c59ee877165dc6931d38c061499f7d66
    25 alphaUSD -> other wallet 0x3676c8a452cda51fc06a521e826e0b3dfc81e39a5dba26998c0be10742881723
    25 alphaUSD -> merchant     0xf026cc1d499f4a0cfb12da492796b823ea0904c41ace7c073a764808756cebaf  <- settled the invoice
   4 transfers carry the memo; only one of them paid the invoice.
```

Walk through the output in this order:

* **Setup and step 1.** The merchant is just an address because it only receives; the customer is a fresh key funded by `client.faucet.fundSync`. `fetchWithRetry` at the top retries requests the public RPC rate-limited (HTTP 429 or a "rate limited" error; such a request never reaches the node), so a run can pause briefly when a whole class runs it at once. A transaction is never re-sent after any other error: while building this demo, one payment's HTTP request failed even though the transaction had landed, which is exactly why the worker trusts chain data, not the sender's view of what happened.
* **Step 2, the invoice.** The invoice lives in the merchant's database, not on-chain. Point at the memo: `INV-…` as ASCII bytes, zero-padded on the right to exactly 32 bytes by `stringToHex(id, { size: 32 })`, because a TIP-20 memo is always a `bytes32`.
* **Step 3, six real transactions.** Open a couple of links. `wrong memo` is missing the last digit; `underpaid`, `wrong token`, and `wrong recipient` all carry the correct memo; `reverted` tries to send more than the customer owns and is pushed through without simulation (`gas` set by hand) so that a real failed transaction exists on-chain. Tempo charges the fee as a separate transfer from the payer, so `correct` delivers the full 25 alphaUSD to the merchant.
* **Step 4, the worker.** Read each line against `reconcile()`. The checks run in order: already processed, receipt status, a `TransferWithMemo` to the merchant, memo maps to an invoice, invoice still open, token, amount. `wrong recipient` is the "I paid, here is my hash" case: a real, successful transfer with the right memo that went to the customer's own other wallet. `webhook retried` is the same hash arriving again.
* **Step 5, the invoice.** Only now is the invoice `paid`, and it stores the hash that paid it, which is your audit trail.
* **Step 6, the indexer.** One SQL query against the public TIDX endpoint finds every `TransferWithMemo` carrying this memo: four rows (the reverted transaction emitted no memo event, and the typo is a different memo), one of which settled the invoice. Use the indexer to find candidates (support tools, finance reports, a polling worker), but credit only after `reconcile()`. It trails the chain head by a few blocks and is rate-limited per IP (the response headers we saw allow 10 requests per minute), so the script polls briefly and the RPC checks never depend on it. The docs also describe a Tempo API route, `https://api.tempo.xyz/v1/indexer/query`, which answered our anonymous requests with `402 Payment Required`; the hosted tidx endpoint used here needs no key.

## 4. Student Exercise

In step 3, change the `wrong memo` payment so it uses the right invoice id, encoded the other way: `pad(stringToHex(invoice.id), { size: 32 })` (add `pad` to the `viem` import). Run the demo again. Steps 3 and 4 now print exactly the same memo text as the invoice, yet the worker still answers `REJECT memo "INV-…" matches no invoice`, and step 6 still lists four transfers. `pad` puts the zero bytes on the left (`0x0000…494e56…`) while the invoice memo has them on the right (`0x494e56…0000`): the chain compares 32 raw bytes, not text. The Tempo docs use both encodings in different examples, so pick one, document it for your payers, and match on bytes.

## Production Takeaway

Keep invoices (expected recipient, token, amount, memo) and every processed payment in your own database, and let one worker turn receipts into business state: a webhook or an indexer row comes in, the receipt is checked, and the invoice update and the processed-payment record are written in the same database transaction. Key processed payments by tx hash plus log index, because one Tempo transaction can batch several transfers, and send every mismatch to a review queue instead of dropping it, because real money arrived. If memos are too error-prone for your payers, virtual addresses give each customer a distinct deposit address that still lands in your master wallet; you still verify the receipt, token, and amount before crediting.

## References

* [How to receive stablecoin payments](https://tempo.xyz/developers/docs/guide/payments/accept-a-payment)
* [Attach a Transfer Memo](https://tempo.xyz/developers/docs/guide/payments/transfer-memos)
* [How to send a stablecoin payment on Tempo](https://tempo.xyz/developers/docs/guide/payments/send-a-payment)
* [TIP-20 Tokens Specification](https://tempo.xyz/developers/docs/protocol/tip20/spec)
* [Tempo Indexer API](https://tempo.xyz/developers/docs/api/indexer-api)
* [tidx: hosted endpoints, tables, and event signatures](https://github.com/tempoxyz/tidx)
* [Use virtual addresses for deposits](https://tempo.xyz/developers/docs/guide/payments/virtual-addresses)
* [viem: token.transfer](https://viem.sh/tempo/actions/token.transfer)
