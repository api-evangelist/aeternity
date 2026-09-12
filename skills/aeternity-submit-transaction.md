---
name: aeternity-submit-transaction
description: Sign and broadcast a transaction to the æternity chain, rehearse it first with dry-run, and confirm inclusion — the only chain-state write this API exposes.
api: Aeternity Node API
base_url: https://mainnet.aeternity.io/v3
generated: '2026-09-12'
method: generated
source: openapi/aeternity-node-openapi.yml
operations:
  - GetStatus
  - GetAccountByPubkey
  - GetAccountNextNonce
  - ProtectedDryRunTxs
  - PostTransaction
  - GetPendingAccountTransactionsByPubkey
  - GetTransactionByHash
  - GetTransactionInfoByHash
---

# Submit a transaction to æternity

There is exactly one operation on this API that changes chain state: `PostTransaction`
(`POST /transactions`). Everything else reads. Read this whole file before you call it — once a
transaction is included in a micro block it is **final**, and no refund, void, cancel or reverse
operation exists anywhere in this contract.

## Before you start

- **No credentials.** The node API declares no securityScheme and needs none. Authority comes from the
  signature inside the payload you post, made with the sender's own private key.
- **Signing is your job, not the node's.** The node never holds a key. Serialize and sign locally, or use
  `@aeternity/aepp-sdk` (npm, 15.0.0) which does both.
- **Rehearse on testnet first.** `https://testnet.aeternity.io/v3` runs the identical node release with
  network_id `ae_uat`; get test coins at https://faucet.aepps.com/. A transaction signed over `ae_uat`
  is cryptographically invalid on `ae_mainnet`, so testnet work cannot leak into production.

## Steps

1. **Check the node is answering and which chain you are on.**
   `GetStatus` (`GET /status`) returns `network_id` (`ae_mainnet` or `ae_uat`), `node_version` and the
   active protocol ladder. Confirm `network_id` matches the network you intend to sign over.

2. **Read the sender's state.**
   `GetAccountByPubkey` (`GET /accounts/{pubkey}`) for the balance. `{pubkey}` is an `ak_`-prefixed
   address; a malformed prefix returns 400, not 404.

3. **Get the nonce. Always, immediately before signing.**
   `GetAccountNextNonce` (`GET /accounts/{pubkey}/next-nonce`). The nonce is what makes this write
   replay-safe: there is no `Idempotency-Key` header on this API, and the protection you get instead is
   that a nonce can be consumed only once. Never cache a nonce across attempts, and never reuse one
   nonce for two different payloads — the second is simply invalid, but an in-flight first attempt you
   have forgotten about is not.

4. **Rehearse with dry-run before anything expensive.**
   `ProtectedDryRunTxs` (`POST /dry-run`) executes unsigned transactions on top of a given block and
   returns the result and a gas estimate, without submitting anything. Use it for every contract call.
   Documented limits: all transaction types are supported except `GAMetaTx`, `PayingForTx` and
   `OffchainTx`; gas defaults to 1,000,000 when unset; total gas per request is capped by a
   node-configured limit and exceeding it returns **403 "Over the gas limit"**. The reported figure is an
   estimate — never below the on-chain cost at the currently activated protocol, because each repriced
   FATE storage register read is floored at that charge.

5. **Sign, then post.**
   `PostTransaction` (`POST /transactions`) takes the signed, `tx_`-encoded transaction and returns its
   `th_` hash. A 400 here means the node rejected the transaction as invalid — a malformed payload, a
   consumed nonce, insufficient balance. **Do not retry a 400 unchanged**; it will fail identically.

6. **Confirm inclusion.**
   `GetPendingAccountTransactionsByPubkey` (`GET /accounts/{pubkey}/transactions/pending`) while it is
   still in the mempool, then `GetTransactionByHash` (`GET /transactions/{hash}`) once mined. For a
   contract call, `GetTransactionInfoByHash` (`GET /transactions/{hash}/info`) returns the call object,
   gas used and emitted events.

## Reading the response, not just the status code

- **`X-Ae-Height` is on every response, including errors.** It is the height of the answering node's chain
  top. A 404 from `GetTransactionByHash` for a transaction you just posted usually means *this node is
  behind*, not that the transaction is gone — compare `X-Ae-Height` against the height you last observed
  before you conclude anything. Resolution is one generation, so it will not move within a generation.
- **503 can come back from any operation.** Overload protection is declared on all 79 operations. The
  response carries `Retry-After` in integer seconds. Honour it; back off exponentially on top of it. This
  is the only throttling signal this API publishes — there are no `X-RateLimit-*` headers.
- **410 Gone means pruned, not absent.** `GetTransactionInfoByHash`,
  `GetAccountByPubkeyAndHeight` and `GetTokenSupplyByHeight` return 410 when the node has
  garbage-collected the state needed. Re-ask the middleware (`https://mainnet.aeternity.io/mdw/v3`),
  which retains history.
- Error bodies are `{"reason": "...", "error_code": "..."}` — a flat vendor object, not RFC 9457
  problem+json. There is no published error-code registry.

## The one thing you cannot undo

Once included, the transfer, the contract call and the gas spent are permanent. The only reversal path in
this contract is `DeleteTxFromMempool` (`DELETE /node/operator/mempool/hash/{hash}`), and it is narrower
than it looks: it is a node-operator endpoint that is **not exposed on the public mainnet gateway**, it
works only while the transaction is still pending, and it removes the transaction from *that one node's*
mempool — other nodes still hold it. Treat every `PostTransaction` as irreversible and confirm the payload
before you sign it.
