---
name: aeternity-watch-chain-events
description: Subscribe to live æternity chain events over the middleware WebSocket — key blocks, micro blocks, transactions, or every transaction touching one account, contract, oracle or name.
api: Aeternity Middleware API
base_url: wss://mainnet.aeternity.io/mdw/v3/websocket
generated: '2026-09-12'
method: generated
source: asyncapi/aeternity-middleware-websocket-asyncapi.yml, https://github.com/aeternity/ae_mdw/blob/master/README.md#websocket-interface
operations:
  - subscribe
  - unsubscribe
  - ping
---

# Watch æternity chain events live

æternity publishes no webhooks. The push surface is a WebSocket subscription stream on the middleware:
`wss://mainnet.aeternity.io/mdw/v3/websocket` (testnet: `wss://testnet.aeternity.io/mdw/v3/websocket`). No
credential is required. The full channel and message contract is in
`asyncapi/aeternity-middleware-websocket-asyncapi.yml`.

## Subscribing

Send `{"op":"Subscribe","payload":"<channel>"}` where channel is one of:

| Channel | What arrives |
|---|---|
| `KeyBlocks` | every new key block |
| `MicroBlocks` | every new micro block |
| `Transactions` | every transaction |
| `Object` | every transaction referencing one entity — add `"target":"<id>"` |

`Object` takes any æternity identifier as its target — an account (`ak_`), oracle (`ok_`), contract
(`ct_`), name (`nm_`) or channel (`ch_`). An oracle operator subscribes to their own `ok_` id to be told
about incoming queries; a wallet subscribes to an `ak_` id to follow one user.

Unsubscribe with `{"op":"Unsubscribe","payload":"<channel>"}`.

## Two rules that will bite you

1. **Every event is delivered twice.** Once when the node has synced the block or transaction, and again
   once AeMdw has indexed it. The `source` field on the published message says which: `"node"` or `"mdw"`.
   Deduplicate on `source`, or you will double-count. Only `"mdw"` messages are guaranteed to be queryable
   through the middleware REST/GraphQL surface at that moment.

2. **Replies are lean, as of ae_mdw 1.105.0.** A Subscribe or Unsubscribe reply is a single-element list
   naming only the channel just added or removed — it is *not* your full subscription list. Code written
   against the older behaviour that reads the reply as the complete set will silently lose track. The
   `WS_SUBS_FULL_LIST_REPLY` flag restores the old shape and is explicitly slated for removal; do not
   depend on it.

## Keeping the connection alive and checking state

Send `{"op":"Ping"}` — no payload field — roughly **every 10 minutes**. The server idle timeout and most
reverse proxies close silent connections, and the Pong counts as traffic on both sides. The reply is
`{"subscriptions":[...],"count":N,"payload":"Pong"}` where `count` is the true total and `subscriptions`
is a **sample** of up to 1000 entries, with `"has_more":true` when truncated. There is no cursor and no
offset: the full subscription list cannot be enumerated. Track your own subscriptions locally as you make
them and use `count` to verify the total matches.

## Limits and reconnection

Documented defaults: 1000 total concurrent connections, **50 per IP**, 100,000 subscriptions per
connection, 2,000,000 total, 2000 queued messages per client before shedding. Connections exceeding a
limit are rejected **at the handshake** and the server closes with a normal closure — so handle `CLOSE`
frames and apply exponential back-off rather than reconnecting in a tight loop. A normal closure here
means "you were refused", not "nothing is wrong".

The legacy V1 stream at `wss://mainnet.aeternity.io/mdw/websocket` is still served; it takes the same
commands and differs only in how mdw-sourced payloads are rendered. Prefer V3, where an mdw-sourced payload
is the same object the `/mdw/v3` REST endpoint returns.
