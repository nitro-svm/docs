# API Reference

This is the wire reference for clients that don't use the Rust client. Rust users get the same protocol through [`simulator-client`](../rust-reference.md); its types are in [`simulator_api` on docs.rs](https://docs.rs/simulator-api/latest/simulator_api/). Everything here matches `simulator-api` 0.29.

## Surfaces

The server is `simulator.termina.technology`. It exposes four surfaces:

| Surface | Endpoint | API key | Protocol |
| --- | --- | --- | --- |
| The session control channel | `wss://simulator.termina.technology/backtest` | Required | Custom JSON over WebSocket. See [Session Control Channel](session-control-channel.md). |
| The session's RPC endpoint | `https://simulator.termina.technology/backtest/<session_id>` | Not used | Solana JSON-RPC over HTTP. See [Session RPC and Subscriptions](session-rpc.md). |
| Session subscriptions | `wss://simulator.termina.technology/backtest/<session_id>` | Not used | Solana-style subscriptions over WebSocket. See [Session RPC and Subscriptions](session-rpc.md#subscriptions). |
| HTTP endpoints | `https://simulator.termina.technology/<path>` | Required (optional on `/available-ranges`) | JSON over HTTP. See [HTTP Endpoints](http-endpoints.md). |

New to the protocol? Start with the [Raw WebSocket Walkthrough](raw-websocket-walkthrough.md): one session from create to close, then a reattach.

A **batch** is a group of a slot's transactions that execute together; `batchIndex` numbers them within the slot. A **parallel run** is one range split across several sessions.

## Authentication

Send your API key as the `X-API-Key` header, on the WebSocket upgrade of the session control channel and on every HTTP request.

* A missing or invalid key gets HTTP `401` with the body `Unauthorized: Invalid or missing API key`. Retrying doesn't help.
* The key goes in a header only. A browser `WebSocket` can't set headers, so connect from a backend.
* The session's RPC endpoint and its subscriptions take no key. The session ID is the credential, so keep it private.

## Field Casing

Casing differs by where a field is declared. The table below is the contract: send exactly these spellings.

| Where | Casing | Examples |
| --- | --- | --- |
| Method names, `status` values, error names | camelCase | `createBacktestSession`, `startingRuntime`, `bundleNotFound` |
| Fields of a `params` object (create, `continue`, `continueTo`, the `*V1` requests, `paused`, `discoveryBatch`, `summary`) | camelCase | `startSlot`, `advanceCount`, `batchIndex`, `sessionId` |
| The sequence number on a response | camelCase | `seqId` |
| Fields declared directly on a request or response variant | snake_case | `session_id`, `last_sequence`, `rpc_endpoint`, `control_session_id`, `seq_id` (inside `sessionEventV2`) |
| Fields of an `error` | snake_case | `start_slot`, `session_id` |
| HTTP request and response bodies, and their enum values | snake_case | `bundle_start_slot`, `request_id`, `in_progress`, `account_state` |
| `accountDiffNotification` results | snake_case | `tx_index`, `block_time` |
| Every other subscription result | camelCase | `actionIndex`, `transactionOutcomes` |
| Aggregator names | lowercase | `jupiter`, `okx`, `titan`, `dflow` |

{% hint style="warning" %}
The server ignores fields it doesn't know. A misspelled optional field, such as `send_summary` instead of `sendSummary`, does nothing and raises no error.
{% endhint %}

## Encoded Fields

| Field | Encoding |
| --- | --- |
| Addresses, program IDs, mints | base58 strings |
| Signatures | base58 strings |
| Session IDs | `backtest_` followed by 32 hex characters |
| Parallel run IDs | `parallel_` followed by 32 hex characters |
| User transactions (`continue.transactions`, `sendTransaction`, `simulateTransaction`) | base64 of the signed transaction's standard wire bytes |
| Account data in an account override | `{"data": "<base64>", "encoding": "base64"}` |
| `actions` in `createBacktestSession` | One string: base64 of the zstd-compressed JSON array of actions |
| `transactions` and `slots` inside an action | One string each: base64 of the zstd-compressed JSON array |
| `discoveryBatch.transactions`, `actionNotification.matched`, a reroute's routed transaction | `{"data": "<base64>", "encoding": "base64"}` holding a wincode-serialized `VersionedTransaction`. Rust decodes it with `simulator_client::decode_transaction`. |
| Program binaries in `POST /generate-depth` | base64 |
| `getAccountInfo` and `getProgramAccounts` account data | `base64+zstd` unless you pass `encoding` |
| Times in HTTP bodies | RFC 3339 UTC strings, or Unix milliseconds in fields named `*_unix_ms` |

An account override is the whole account:

```json
{
  "data": { "data": "<base64>", "encoding": "base64" },
  "executable": false,
  "lamports": 1000000,
  "owner": "11111111111111111111111111111111",
  "space": 0
}
```

* `space` is the decoded length of `data`.
* The session sets `rentEpoch` to 0.
* Overrides in scheduled actions, depth jobs and `simulateTransaction` reject an executable account or one owned by a program loader. To change a program, use a program override.

## Versions

| Package | Version |
| --- | --- |
| `simulator-api` | 0.29 |
| `simulator-client` | 0.29 |
| `sim-cli` | 0.22 |
