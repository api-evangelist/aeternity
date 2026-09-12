---
name: aeternity-read-account-history
description: Read everything the chain knows about an æternity account — balance, unified activity feed, AEX-9 token balances, AEX-141 NFTs, name claims and DEX swaps — using the middleware index rather than the node.
api: Aeternity Middleware API
base_url: https://mainnet.aeternity.io/mdw/v3
generated: '2026-09-12'
method: generated
source: openapi/aeternity-middleware-openapi.yml, graphql/aeternity-middleware.graphql
operations:
  - GetStatus
  - GetAccountActivities
  - GetAccountTransactionsCount
  - GetAex9AccountBalances
  - GetAex141OwnedTokens
  - GetAccountNameClaims
  - GetAccountPointees
  - GetAccountDexSwaps
---

# Read an account's history

Use the **middleware**, not the node, for anything historical. The node answers about current state and
prunes old state (three operations return 410 Gone once it has); ae_mdw keeps the index. It is
unauthenticated, CORS-open, and read-only.

## Steps

1. **Check the index is caught up first.** `GetStatus` (`GET /status`) returns `mdw_synced`,
   `mdw_syncing`, `mdw_height` and `node_height`. If `mdw_height` is behind `node_height`, anything newer
   than `mdw_height` is simply not indexed yet and will 404. Check this before you conclude a resource
   does not exist.

2. **The unified feed is the best starting point.** `GetAccountActivities`
   (`GET /accounts/{accountId}/activities`) returns transactions, transfers, swaps, claims and token
   events for one `ak_` address in one stream, filterable by type
   (`TRANSACTIONS`, `TRANSFERS`, `SWAPS`, `CLAIMS`, `AEX9`, `AEX141`, `AEXN`, `CONTRACT`).

3. **Counts and token positions.**
   - `GetAccountTransactionsCount` (`GET /accounts/{accountId}/transactions/count`)
   - `GetAex9AccountBalances` (`GET /accounts/{accountId}/aex9/balances`) — fungible tokens (AEX-9)
   - `GetAex141OwnedTokens` (`GET /accounts/{accountId}/aex141/tokens`) — NFTs (AEX-141)
   - `GetAccountDexSwaps` (`GET /accounts/{accountId}/dex/swaps`) — Superhero DEX activity

4. **Names.** `GetAccountNameClaims` (`GET /accounts/{accountId}/names/claims`) for names this account
   claimed; `GetAccountPointees` (`GET /accounts/{accountId}/names/pointees`) for names pointing *at* it.

## Pagination

Every list endpoint is cursor-paginated: pass `limit` and `direction`, and follow `next` from the response
envelope `{data, next, prev}`. **Cursors are opaque** — replay the `next` value verbatim; do not construct
one, and do not assume it survives a re-index.

## When REST cannot answer it

The same data is served over GraphQL at `https://mainnet.aeternity.io/mdw/graphql`, introspection open, 85
query fields. Eleven of them have no REST equivalent, and this is where you go for them — chain-wide
`aex141Transfers`, `nameHistory`, `searchNames`, `contracts`, `channelUpdates`, `wealth`,
`topMiners24hStats`. The full mapping is in `mcp/aeternity-tool-crosswalk.yml`. Going the other way,
Hyperchains (`/hyperchain/epochs`, `/hyperchain/schedule`, `/hyperchain/validators`) is REST-only — the
GraphQL schema has no hyperchain type. Query complexity is capped at 1000 and successful GraphQL responses
are cached for 5 seconds.

## Errors

`{"error": "..."}` on 400 (bad id, bad cursor, bad range) and 404 (not indexed). Not RFC 9457. A 400 on an
identifier is nearly always a wrong prefix — this chain's ids are self-describing (`ak_` account,
`th_` transaction, `ct_` contract, `nm_` name, `ok_` oracle, `ch_` channel), so validate the prefix
client-side before calling.
