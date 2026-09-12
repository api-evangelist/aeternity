---
name: aeternity-inspect-token-contract
description: Inspect an æternity AEX-9 fungible token or AEX-141 NFT contract — holders, balances, balance history, tokens, templates and transfers — using the middleware's standard-aware endpoints.
api: Aeternity Middleware API
base_url: https://mainnet.aeternity.io/mdw/v3
generated: '2026-09-12'
method: generated
source: openapi/aeternity-middleware-openapi.yml, conformance/aeternity-conformance.yml
operations:
  - GetSortedAex9Tokens
  - GetAex9TokensCount
  - GetAex9ByContract
  - GetAex9ContractBalances
  - GetAex9ContractAccountBalance
  - GetAex9ContractAccountBalanceHistory
  - GetSortedAex141Contracts
  - GetAex141ByContract
  - GetAex141ContractTokens
  - GetAex141TokenOwner
  - GetAex141ContractTemplates
  - GetAex141TemplateTokens
  - GetAex141ContractTransfers
  - GetAex9TransfersStats
---

# Inspect a token contract

æternity's token standards are its own: **AEX-9** for fungible tokens (the ERC-20 analogue) and
**AEX-141** for NFTs (the ERC-721 analogue, with a template extension). They are published through the
æternity Expansions process, and — unusually — they are first-class resources in the API contract itself
rather than something you decode from raw contract calls. That is the point of this skill: you do not need
to know Sophia or call the contract to read a token's state.

## Fungible tokens (AEX-9)

- `GetSortedAex9Tokens` (`GET /aex9`) — browse every indexed AEX-9 contract, sortable.
- `GetAex9TokensCount` (`GET /aex9/count`) — how many exist.
- `GetAex9ByContract` (`GET /aex9/{id}`) — one token's metadata by `ct_` id.
- `GetAex9ContractBalances` (`GET /aex9/{contractId}/balances`) — the holder set.
- `GetAex9ContractAccountBalance` (`GET /aex9/{contractId}/balances/{accountId}`) — one holder.
- `GetAex9ContractAccountBalanceHistory` (`GET /aex9/{contractId}/balances/{accountId}/history`) — that
  holder's balance over time. This is the endpoint you cannot get from the node, which keeps no history.
- `GetAex9TransfersStats` (`GET /stats/aex9-transfers`) — chain-wide transfer volume.

Chain-wide AEX-9 transfers and per-contract transfer lists have **no REST endpoint** — use the
`aex9ContractTransfers` field on the GraphQL surface instead (see `mcp/aeternity-tool-crosswalk.yml`).

## NFTs (AEX-141)

- `GetSortedAex141Contracts` (`GET /aex141`) and `GetAex141ByContract` (`GET /aex141/{id}`).
- `GetAex141ContractTokens` (`GET /aex141/{contractId}/tokens`) and `GetAex141TokenOwner`
  (`GET /aex141/{contractId}/tokens/{tokenId}`).
- `GetAex141ContractTemplates` (`GET /aex141/{contractId}/templates`) and `GetAex141TemplateTokens`
  (`GET /aex141/{contractId}/templates/{templateId}/tokens`) — the template extension, where many tokens
  share one metadata template.
- `GetAex141ContractTransfers` (`GET /aex141/{contractId}/transfers`).

For one account's holdings rather than one contract's, use `GetAex141OwnedTokens`
(`GET /accounts/{accountId}/aex141/tokens`) and `GetAex9AccountBalances`
(`GET /accounts/{accountId}/aex9/balances`).

## Correctness notes

- **Balances are re-derived, not accumulated.** Since ae_mdw 1.107.2 the index re-derives AEX-9 balances
  from a dry-run rather than doing per-event arithmetic, and skips event balance updates for tokens
  created inside the same call. If you are comparing against your own event-summed figures and they
  disagree, the index is the one that was fixed.
- **Check `mdw_synced` / `mdw_height` on `GET /status`** before treating a 404 as "this token does not
  exist". A contract deployed above `mdw_height` is not indexed yet.
- Every list is cursor-paginated (`limit`, `direction`, opaque `cursor`, `{data, next, prev}` envelope).
- These endpoints are read-only. Minting, transferring or burning is a contract call, which means signing
  a `ContractCallTx` and posting it through the node — see the `aeternity-submit-transaction` skill, and
  note that it is irreversible.
