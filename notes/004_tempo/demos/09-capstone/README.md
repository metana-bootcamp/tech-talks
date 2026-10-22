# Demo 09: Capstone Payment App

A merchant issues two invoices on Tempo testnet and a customer pays them. The app sponsors the fee only for the exact payment of an open invoice, and its backend marks an invoice paid only after reading the payment back from the chain. Along the way the customer also sends payments the chain accepts but the invoice does not. The lesson: the transaction is not the product state. Sponsorship decides what the app will pay for, and reconciliation decides what counts as paid.

## Questions This Demo Answers

* What is the success moment, the transaction or the invoice? The invoice, after reconciliation. In step 5 two transactions succeed on-chain and INV-2002 is still open.
* Where does sponsorship fit? Before the sponsor signs anything. The app's relay checks the request against invoice state (an open invoice, the merchant, alphaUSD, the exact amount) and only then co-signs. The three refusals in step 4 never reach the chain.
* Where does reconciliation fit? After the transaction exists and before any product state changes. The backend fetches the receipt by hash from the chain and checks the `TransferWithMemo` event's token, recipient, amount and memo against an open invoice.
* Why record transaction hashes when there is already a memo? Notifications repeat (webhooks retry and indexers replay). Crediting by hash makes reconciliation idempotent, so the second notice in step 3 is ignored. Paying an invoice twice is stopped earlier, by the sponsorship policy (step 4).
* Can the app stop customers from sending wrong payments? No. Anyone can send any token, with any memo, to the merchant's address (step 5). The app can only decline to sponsor and decline to credit them. The money still arrives, so a real product needs a refund or review flow (step 7).
* What would a production app add? Durable storage for invoices and credited hashes, a webhook or the indexer to learn about payments, authentication and rate limits on the relay, and refunds. Everything on-chain here is already real.

## 1. Create The Project

```bash
mkdir tempo-demo-09-capstone
cd tempo-demo-09-capstone
npm init -y
npm pkg set type=module
npm install viem@2.57.2
npm install --save-dev tsx typescript @types/node
mkdir src
```

No `.env` is needed: the script creates throwaway testnet accounts and funds them from the Tempo faucet on every run.

## 2. Write The Script

The relay is the same one as in Demo 02, and reconciliation follows Demo 03.

Create `src/paymentApp.ts`:

```ts
import { createServer, type Server } from "node:http";
import type { AddressInfo } from "node:net";
import {
  type Address,
  type BaseError,
  type Hash,
  type Hex,
  decodeFunctionData,
  formatUnits,
  isAddressEqual,
  parseEventLogs,
  parseUnits,
  stringToHex,
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
const betaUsd = "0x20c0000000000000000000000000000000000002";
const explorer = "https://explore.testnet.tempo.xyz/tx/";
const usd = (amount: bigint) => formatUnits(amount, 6);

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
const customer = Account.fromSecp256k1(generatePrivateKey());
const sponsor = Account.fromSecp256k1(generatePrivateKey()); // the app's fee payer
const merchant = Account.fromSecp256k1(generatePrivateKey()).address;
const stranger = Account.fromSecp256k1(generatePrivateKey()).address;

// ---- App state: invoices (keyed by their 32-byte memo) and transactions already credited ----
type Invoice = { id: string; memo: Hex; token: Address; amount: bigint; status: "open" | "paid"; paidBy?: Hash };
const invoices = new Map<Hex, Invoice>();
const credited = new Set<Hash>();

function createInvoice(id: string, amount: string): Invoice {
  const invoice: Invoice = { id, memo: stringToHex(id, { size: 32 }), token: alphaUsd, amount: parseUnits(amount, 6), status: "open" };
  invoices.set(invoice.memo, invoice);
  return invoice;
}

// ---- Sponsorship policy: pay the fee only for the exact payment of an open invoice ----
function sponsorPolicy(tx: Transaction.TransactionRequest): Relay.feePayer.Validation {
  const call = tx.calls?.length === 1 ? tx.calls[0] : undefined;
  if (!call?.to || !call.data || !isAddressEqual(TempoAddress.resolve(call.to), alphaUsd)) return false;
  const { functionName, args } = decodeFunctionData({ abi: Abis.tip20, data: call.data });
  if (functionName !== "transferWithMemo") return false;
  const [to, amount, memo] = args;
  const invoice = invoices.get(memo);
  return invoice?.status === "open" && isAddressEqual(to, merchant) && amount === invoice.amount;
}

// ---- Reconciliation: decide from the chain, not from the client, whether a transaction pays an invoice ----
async function reconcile(hash: Hash): Promise<string> {
  if (credited.has(hash)) return "ignored, this transaction was already credited";
  const receipt = await chain.getTransactionReceipt({ hash });
  if (receipt.status !== "success") return "rejected, the transaction reverted";
  const payment = parseEventLogs({ abi: Abis.tip20, eventName: "TransferWithMemo", logs: receipt.logs }).find(
    (log) => isAddressEqual(log.args.to, merchant),
  );
  const invoice = payment && invoices.get(payment.args.memo);
  if (!payment || !invoice) return "rejected, no payment to the merchant for a known invoice";
  if (invoice.status !== "open") return `rejected, ${invoice.id} is already ${invoice.status}`;
  if (!isAddressEqual(payment.address, invoice.token)) return `rejected, ${invoice.id} must be paid in alphaUSD`;
  if (payment.args.amount < invoice.amount) {
    return `rejected, ${invoice.id} underpaid: ${usd(payment.args.amount)} of ${usd(invoice.amount)}`;
  }
  credited.add(hash);
  Object.assign(invoice, { status: "paid", paidBy: hash });
  return `${invoice.id} PAID`;
}

// ---- The app's relay: viem's built-in Relay with the fee-payer plugin, served over node:http ----
async function startRelay(): Promise<{ url: string; server: Server }> {
  const relay = Relay.create({
    client: chain,
    plugins: [Relay.feePayer({ account: sponsor, feeToken: Addresses.pathUsd, validate: sponsorPolicy })],
  });
  const server = createServer(async (req, res) => {
    let body = "";
    for await (const chunk of req) body += chunk;
    const headers = { "content-type": "application/json" };
    const response = await relay.fetch(new Request("http://relay", { method: "POST", headers, body }));
    res.writeHead(response.status, headers).end(await response.text());
  });
  await new Promise<void>((resolve) => server.listen(0, "127.0.0.1", resolve));
  return { url: `http://127.0.0.1:${(server.address() as AddressInfo).port}`, server };
}

console.log("1. Fund the customer and the app's sponsor, start the app's relay");
await Actions.faucet.fundSync(chain, { account: customer, timeout: 60_000 });
await Actions.faucet.fundSync(chain, { account: sponsor, timeout: 60_000 });
const relay = await startRelay();
const app = createClient({
  account: customer,
  testnet: true,
  transport: withRelay(rpc, http(relay.url, { fetchFn: fetchWithRetry, timeout: 60_000 })),
});
console.log(`   customer ${customer.address}, sponsor ${sponsor.address}`);
console.log(`   merchant ${merchant}, relay ${relay.url}`);

// The customer's wallet: one payment with the invoice memo, sponsored by the app unless feePayer is false
async function pay(invoice: Invoice, options: { amount?: string; to?: Address; token?: Address; feePayer?: boolean } = {}) {
  const { to = merchant, token = invoice.token, feePayer = true } = options;
  const amount = options.amount ? parseUnits(options.amount, 6) : invoice.amount;
  const { receipt } = await app.token.transferSync({ token, to, amount, memo: invoice.memo, feePayer });
  const paidBy = isAddressEqual(receipt.feePayer!, sponsor.address) ? "the app's sponsor" : "the customer";
  console.log(`   tx ${explorer}${receipt.transactionHash} (fee paid by ${paidBy})`);
  return receipt.transactionHash;
}
async function refused(label: string, attempt: () => Promise<unknown>) {
  try {
    await attempt();
    console.log(`   ${label}: UNEXPECTED, the app sponsored it`);
    process.exitCode = 1;
  } catch (error) {
    console.log(`   ${label}: refused (${(error as BaseError).details})`);
  }
}

console.log("\n2. The merchant creates two invoices, payable in alphaUSD");
const first = createInvoice("INV-2001", "40");
const second = createInvoice("INV-2002", "75");
for (const invoice of [first, second]) console.log(`   ${invoice.id}: ${usd(invoice.amount)} alphaUSD, memo ${invoice.memo.slice(0, 18)}…`);

console.log("\n3. The customer pays INV-2001 through the app; the backend reconciles it");
const paid = await pay(first);
console.log(`   reconcile: ${await reconcile(paid)}`);
console.log(`   same notification again: ${await reconcile(paid)}`);

console.log("\n4. Requests the app refuses to sponsor");
await refused("pay INV-2001 a second time", () => pay(first));
await refused("pay a stranger with the INV-2002 memo", () => pay(second, { to: stranger }));
await refused("pay 50 of the 75 alphaUSD due on INV-2002", () => pay(second, { amount: "50" }));

console.log("\n5. Payments the chain accepts but the invoice does not (the customer pays the fee)");
console.log("   50 alphaUSD for INV-2002");
console.log(`   reconcile: ${await reconcile(await pay(second, { amount: "50", feePayer: false }))}`);
console.log("   75 betaUSD for INV-2002");
console.log(`   reconcile: ${await reconcile(await pay(second, { token: betaUsd, feePayer: false }))}`);

console.log("\n6. The customer pays INV-2002 correctly");
console.log(`   reconcile: ${await reconcile(await pay(second))}`);

console.log("\n7. Ledger vs wallet");
for (const invoice of invoices.values()) {
  console.log(`   ${invoice.id}  ${usd(invoice.amount).padStart(5)} alphaUSD  ${invoice.status}  ${invoice.paidBy ?? ""}`);
}
const alphaIn = (await chain.token.getBalance({ token: alphaUsd, account: merchant })).amount;
const betaIn = (await chain.token.getBalance({ token: betaUsd, account: merchant })).amount;
console.log(`   merchant wallet: ${usd(alphaIn)} alphaUSD + ${usd(betaIn)} betaUSD`);
const unmatched = alphaIn - first.amount - second.amount;
console.log(`   invoices explain ${usd(first.amount + second.amount)} alphaUSD; ${usd(unmatched)} alphaUSD and ${usd(betaIn)} betaUSD need a refund or review`);

relay.server.close();
if ([...invoices.values()].some((invoice) => invoice.status !== "paid")) process.exitCode = 1;
```

## 3. Run The Demo

```bash
npx tsx src/paymentApp.ts
```

Output from a real run (addresses and hashes will differ):

```text
1. Fund the customer and the app's sponsor, start the app's relay
   customer 0x92563769eDaf2e2b77E09D95C179104Da80e45d2, sponsor 0x690b383702458Ee4F2Cd3709181B6Ea8E58D696e
   merchant 0xeA02eFB1585B2c386f2Dc97Cba3821278dBd4c43, relay http://127.0.0.1:38837

2. The merchant creates two invoices, payable in alphaUSD
   INV-2001: 40 alphaUSD, memo 0x494e562d32303031…
   INV-2002: 75 alphaUSD, memo 0x494e562d32303032…

3. The customer pays INV-2001 through the app; the backend reconciles it
   tx https://explore.testnet.tempo.xyz/tx/0x3d4f6e260fa45bc6d433825302f480bf08c8e8a791dae5a8cc8f14d1da818715 (fee paid by the app's sponsor)
   reconcile: INV-2001 PAID
   same notification again: ignored, this transaction was already credited

4. Requests the app refuses to sponsor
   pay INV-2001 a second time: refused (Sponsorship rejected.)
   pay a stranger with the INV-2002 memo: refused (Sponsorship rejected.)
   pay 50 of the 75 alphaUSD due on INV-2002: refused (Sponsorship rejected.)

5. Payments the chain accepts but the invoice does not (the customer pays the fee)
   50 alphaUSD for INV-2002
   tx https://explore.testnet.tempo.xyz/tx/0x986aa687d8bae9480fd329093d658fb4727c3c08efc3cd2b23c3dac2e4c9993f (fee paid by the customer)
   reconcile: rejected, INV-2002 underpaid: 50 of 75
   75 betaUSD for INV-2002
   tx https://explore.testnet.tempo.xyz/tx/0xb1ab9c871604d667fde5f4232b3a287932071ba45e64a27654b375fbf75e0abc (fee paid by the customer)
   reconcile: rejected, INV-2002 must be paid in alphaUSD

6. The customer pays INV-2002 correctly
   tx https://explore.testnet.tempo.xyz/tx/0xbf912287403726e3c6bb4968e228e0c44dc403e66aea3e8e5a9a0e4daec14e8e (fee paid by the app's sponsor)
   reconcile: INV-2002 PAID

7. Ledger vs wallet
   INV-2001     40 alphaUSD  paid  0x3d4f6e260fa45bc6d433825302f480bf08c8e8a791dae5a8cc8f14d1da818715
   INV-2002     75 alphaUSD  paid  0xbf912287403726e3c6bb4968e228e0c44dc403e66aea3e8e5a9a0e4daec14e8e
   merchant wallet: 165 alphaUSD + 75 betaUSD
   invoices explain 115 alphaUSD; 50 alphaUSD and 75 betaUSD need a refund or review
```

The whole run takes about 10 seconds. Walk through the output in this order:

* **Step 1:** the app has its own sponsor account, funded with pathUSD, and its own relay. The customer's client sends transactions to the chain and sponsorship requests to the relay (`withRelay`).
* **Step 2:** an invoice is just app state: an amount, a token, and a 32-byte memo derived from its ID. Nothing is on-chain yet.
* **Step 3:** the customer pays with the invoice memo, and the receipt shows the app's sponsor paid the fee. The backend reconciles from the receipt it fetches itself, not from anything the client reports. The same notification arriving again changes nothing.
* **Step 4:** the policy reads invoice state. It refuses a second payment of a paid invoice, a payment to anyone but the merchant, and a partial payment. These are refusals from the app's relay, so there is no transaction and no fee.
* **Step 5:** the customer can still pay the fee and send whatever they like. Both transactions succeed on-chain, and reconciliation rejects both: one is short, the other is in the wrong token. INV-2002 stays open.
* **Step 6:** the exact payment is sponsored and reconciles. INV-2002 is paid.
* **Step 7:** the ledger and the wallet disagree. The merchant received 165 alphaUSD and 75 betaUSD, but invoices explain only 115 alphaUSD. The rest is real money that needs a refund or a person to review it.

## 4. Student Exercise

In `sponsorPolicy`, delete the exact-amount check so the last line reads:

```ts
return invoice?.status === "open" && isAddressEqual(to, merchant);
```

Predict what changes, then run it. The partial payment in step 4 is now sponsored (`UNEXPECTED, the app sponsored it`), and the script exits with code 1. The app paid the fee for a payment that settles nothing: reconciliation would still reject it, but the merchant's wallet now holds 215 alphaUSD, with 100 alphaUSD that no invoice explains. A sponsorship policy that is looser than the reconciliation rules pays for payments the business will have to refund.

## Production Takeaway

A payment product is a state machine on top of the chain. Sponsor only the requests that move an invoice toward paid, and credit only what a receipt proves. Treat everything else (duplicates, partial payments, wrong tokens) as a normal case with its own path: refund, review, or ignore. On Tempo, transaction controls, sponsorship, memos, access keys, policies, the DEX and zones are all building blocks for that state machine. None of them replaces it.

## References

* [Sponsor user fees with `feePayer`](https://tempo.xyz/developers/docs/guide/payments/sponsor-user-fees)
* [Tempo Fee Payer API](https://tempo.xyz/developers/docs/api/fee-payer)
* [viem: Run a Relay](https://viem.sh/tempo/guides/relay/run)
* [viem: Fee Payer Plugin](https://viem.sh/tempo/relay/plugins/fee-payer)
* [Attach a transfer memo](https://tempo.xyz/developers/docs/guide/payments/transfer-memos)
* [How to receive stablecoin payments](https://tempo.xyz/developers/docs/guide/payments/accept-a-payment)
* [Tempo indexer API](https://tempo.xyz/developers/docs/api/indexer-api)
