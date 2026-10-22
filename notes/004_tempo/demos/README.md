# Tempo Demo Tutorials

Hands-on demos for the Tempo tech-talk series. Each class has one README that runs live against Tempo's public testnet (Moderato), and students can follow it again afterwards. Every script creates throwaway accounts, funds them from the testnet faucet, and shows the happy path and the failure cases on the real chain. Nothing is simulated.

## Before Class

* You need Node.js 20 or newer. You don't need a wallet, a private key, or a `.env` file.
* Each demo is its own small TypeScript project, and its README lists the exact commands. The demos pin `viem@2.57.2`, the version they were tested with. Demo 08 pins `viem@2.54.6` on purpose, because Tempo's shared testnet zones run an earlier deployment (see its README).
* Run the demo yourself shortly before class. Testnet state changes: DEX prices, other people's orders, and network upgrades. T12 went live on testnet on October 8, 2026 (mainnet follows on October 13), and all the demos were re-run after it.

| Network | Value |
| --- | --- |
| Name | Tempo Testnet (Moderato) |
| Chain ID | `42431` |
| RPC | `https://rpc.moderato.tempo.xyz` |
| Explorer | `https://explore.testnet.tempo.xyz` |
| Faucet | `tempo_fundAddress` (`Actions.faucet.fundSync` in viem): 1,000,000 each of pathUSD, alphaUSD, betaUSD and thetaUSD per call |

## Tutorials

| Class | Tutorial | What students see on testnet | Deck | Runs in |
| --- | --- | --- | --- | --- |
| 01 | [Transaction Controls](01-transaction-controls/README.md) | Payment time windows enforced by the node, two workers colliding on one nonce, parallel `nonceKey` lanes, and an expiring nonce whose replay is refused | 004 | ~20 s + ~10 s |
| 02 | [Sponsored Payments](02-sponsored-payments/README.md) | Tempo's public fee payer, then the app's own relay that sponsors only what its policy allows; the chain rejects replay, expiry and an empty sponsor | 005, 009 | ~35 s |
| 03 | [Payment Reconciliation](03-reconciliation/README.md) | Six real payments against one invoice; a worker reads receipts and memo events and accepts exactly one; the indexer finds them by memo | 002, 003, 009 | ~10 s |
| 04 | [Access Keys And Agents](04-access-keys-agents/README.md) | An agent key with scopes, a budget and an expiry; the protocol refuses every overreach, then revocation and expiry lock it out | 004 | ~35 s |
| 05 | [Batch Payments](05-batch-payments/README.md) | A validated payroll paid as one Tempo Transaction, and an overdrawing batch that fails all or nothing | 004 | ~10 s |
| 06 | [TIP-403 Policies](06-tip403-policies/README.md) | An issuer's own stablecoin behind a whitelist, and a treasury receive policy that parks unwanted deposits in ReceivePolicyGuard until they're recovered | 006 | ~20 s |
| 07 | [Stablecoin DEX](07-stablecoin-dex/README.md) | A quote and a bounded swap, real slippage and no-liquidity reverts, and an atomic swap-and-pay for a merchant who wants another stablecoin | 007 | ~10 s |
| 08 | [Private Zones](08-private-zones/README.md) | Deposits into Zone A, a self-signed RPC token, a private payment, and what each user can and cannot read | 008 | ~10 s |
| 09 | [Capstone Payment App](09-capstone/README.md) | Invoices, sponsorship tied to invoice state, reconciliation from receipts, and the gap between the ledger and the wallet | all | ~10 s |
| 10 | [Fee AMM](10-fee-amm/README.md) | Which token pays each fee and how the proposer gets paid at a fixed 0.997; a new stablecoin refused as a fee token until its Fee AMM pool is seeded | 005 | ~12 s |

The Deck column refers to the slide decks in this folder's parent, for example `004` is [Tempo Transactions](../004_tempo_transactions/index.html).

## Suggested Class Format

Each tutorial is designed for a one-hour class:

```text
0-5 min      Explain the product problem
5-15 min     Show the happy path
15-35 min    Walk through the important steps
35-45 min    Trigger the failure cases
45-55 min    Students do the README's exercise
55-60 min    Review the production takeaway
```

## Notes For Instructors

* Every run uses fresh accounts, so reruns never collide and there's nothing to reset.
* The sample output in each README comes from a real run on October 2, 2026. Addresses, hashes, and some amounts change on every run.
* The public RPC allows a few dozen requests per second per IP, so a whole class on one network gets rate-limited. Each script waits and retries rate-limited requests, which makes runs slower but doesn't break them. In our stress test, all 20 copies running at once from one machine passed. At 30 copies, a few short-lived authorizations (Demos 02 and 04) expired while they waited and needed a rerun.
* The node checks time windows against block timestamps, not your clock. On October 2, 2026, testnet block timestamps ran about 15 minutes ahead of real time for roughly 20 minutes, and time-window steps failed until it recovered. If a step fails with `too close to current time`, compare the latest block timestamp with your clock.
* Never paste a mainnet key into these scripts. They only ever need throwaway testnet keys.
