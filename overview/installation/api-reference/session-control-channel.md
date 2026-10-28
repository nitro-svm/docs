# Session Control Channel

The session control channel creates a session and drives it: you send requests, the server sends responses. One connection drives one session, or one parallel run of sessions.

## Connect

Open a WebSocket to `wss://simulator.termina.technology/backtest` with the `X-API-Key` header.

* The first message must be `createBacktestSession`, `attachBacktestSession` or `attachParallelControlSessionV2`. Anything else gets an `invalidRequest` error.
* Every message is a JSON text frame. The server accepts messages up to 64 MiB. Its responses can be larger than the 1 MiB that some WebSocket libraries accept by default, such as Python's `websockets`: raise that limit, for example with `max_size=2**26`.
* The upgrade fails with HTTP `401` for a missing or unknown key, and `503` while the server is restarting: retry it after a few seconds.
* The server answers WebSocket pings. A connection idle for 4,000 seconds is closed, so ping every 15 to 30 seconds; the Rust client pings every 15.
* Keep the connection open while the session runs. What happens when it drops depends on `disconnectTimeoutSecs`; see [Reattach](#reattach).

## Message Format

Requests and responses share one envelope: the method name, plus `params` when the method carries data.

```json
{ "method": "continue", "params": { "advanceCount": 1 } }
{ "method": "readyForContinue" }
{ "method": "slotNotification", "params": 380000000 }
```

Unless `disconnectTimeoutSecs` is `0`, responses that come from the session carry a sequence number, `seqId`, next to `method`:

```json
{ "seqId": 42, "method": "status", "params": { "status": "executedBlockTransactions" } }
```

* Keep the last `seqId` you received. Reattaching with it replays everything after it.
* The server's own replies carry no `seqId`: `sessionCreated`, `sessionAttached`, `sessionsCreatedV2`, `parallelSessionAttachedV2`, the `success` that answers `resumeAttachedSession` or `closeBacktestSession`, and errors about a request. Parse both shapes.

## Requests

### createBacktestSession

Starts a session over a slot range.

```json
{
  "method": "createBacktestSession",
  "params": {
    "startSlot": 380000000,
    "endSlot": 380000099,
    "sendSummary": true,
    "disconnectTimeoutSecs": 300
  }
}
```

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `startSlot` | integer | required | First slot to replay, inclusive. A single session must start exactly at a range's `bundle_start_slot` from [`GET /available-ranges`](http-endpoints.md#get-available-ranges). |
| `endSlot` | integer | required | Last slot to replay, inclusive. At most the range's `max_bundle_end_slot`. |
| `sendSummary` | boolean | `false` | Attach a session summary to `completed`. |
| `disconnectTimeoutSecs` | integer | `30` (`900` with `parallel`) | Seconds the session waits for you to reattach after the connection drops. Capped at 900. `0` ends the session with the connection and turns off `seqId`. |
| `capacityWaitTimeoutSecs` | integer | `0` | Seconds to keep retrying while the server is at capacity. `0` fails at once when it is busy. |
| `extraComputeUnits` | integer | none | Compute units added to each transaction's `SetComputeUnitLimit`. Only transactions that already set a limit are changed. |
| `discoveries` | array | `[]` | [Discovery filters](#discovery-filters). |
| `actions` | string | none | Scheduled actions: base64 of the zstd-compressed JSON array of [actions](#scheduled-actions). |
| `reroute` | object | none | [Rerouting](#reroute). Absent: plain replay. |
| `replayAccountState` | boolean | `false` | Replay recorded account state instead of executing transactions. Needs an `account_state` bundle for the range. User transactions, `kind: "send"` actions, `getTransaction` and `getSignatureStatuses` are rejected; `simulateTransaction` works. |
| `diagnosticProbes` | array | `[]` | Markets to probe through the router as the session runs, streamed over `diagnosticSubscribe`. See [`DiagnosticProbeParams`](https://docs.rs/simulator-api/latest/simulator_api/struct.DiagnosticProbeParams.html). |
| `parallel` | boolean | `false` | Split the range into one session per bundle and run them at once. See [Parallel Sessions](#parallel-sessions). |
| `name` | string | server-picked | The session's name (or the parallel run's). 1 to 100 letters, digits, `-`, `_` or `.`, starting and ending with a letter or digit. Stored lowercased, never reused. |
| `createToken` | string | none | Any unique string. Resending the same create with the same token after a reconnect reclaims its `name` instead of failing with `sessionNameTaken`. |

The server replies with a `status` of `startingRuntime` about every 10 seconds while the session starts, then `sessionCreated`, `readyForContinue`, and a `slotNotification` with the slot before `startSlot`, which is where the session starts.

* A create that fails gets an `error` and no session. A busy server answers `{"internal": {"error": "unable to start session: insufficient capacity"}}` once `capacityWaitTimeoutSecs` runs out: retry later. There is no per-key limit on sessions.
* If the connection drops before `sessionCreated`, the server abandons the create and no session runs. Send the create again on a new connection, with the same `name` and `createToken` if you set a name.

#### Discovery Filters

A discovery filter makes the session announce a matching batch before it executes, with `discoveryBatch`. Pass its `slot` and `batchIndex` to `continueTo` to pause just before it.

```json
{ "kind": "programExecuted", "value": "<program_id>" }
{ "kind": "oracleUpdate", "value": { "programs": ["<program_id>"], "accounts": ["<address>"] } }
```

* `programExecuted` matches a batch with a transaction that invokes the program.
* `oracleUpdate` matches a market maker's write to its pricing state: a transaction that invokes one of `programs`, moves no token balance and, when `accounts` is set, writes one of them.

#### Scheduled Actions

Each action runs a sequence of transactions automatically, and streams its results over `actionSubscribe`. An action without transactions is a set of account overrides applied from its slot.

```json
{
  "anchor": { "at": "afterEverySlot", "everyNSlots": 10 },
  "kind": "simulate",
  "transactions": "<base64 of zstd(JSON array of base64 transactions)>",
  "accountOverrides": { "<address>": { "data": { "data": "", "encoding": "base64" }, "executable": false, "lamports": 10000000000, "owner": "11111111111111111111111111111111", "space": 0 } },
  "returnAccounts": ["<address>"],
  "label": "probe"
}
```

| Field | Meaning |
| --- | --- |
| `anchor` | Where it fires: `{"at": "beforeSlot", "slots": "<compressed>"}`, `{"at": "afterSlot", "slots": "<compressed>"}`, `{"at": "beforeEverySlot", "everyNSlots": n}`, `{"at": "afterEverySlot", "everyNSlots": n}`, `{"at": "beforeMatch", "filter": <discovery filter>}` or `{"at": "afterMatch", "filter": <discovery filter>}`. With `slots`, transaction `i` fires at `slots[i]`. |
| `kind` | `simulate` (default): runs without committing. `send`: commits, so later slots see the writes. |
| `transactions` | The transactions, compressed as the table says. |
| `accountOverrides` | Account overrides the first transaction sees. Not allowed with `send`. |
| `feedsReroute` | Feed the action's final account state into rerouting from its slot. Needs a `beforeSlot` anchor. |
| `returnAccounts` | Accounts whose state after the last transaction is returned in the result. |
| `label` | Echoed on each result. |
| `resolve` | Rewrites the transactions from live state each time it fires. See [`TxResolver`](https://docs.rs/simulator-api/latest/simulator_api/enum.TxResolver.html). |

#### Reroute

Presence turns rerouting on. `{}` reroutes Jupiter's swaps via an aggregator, for every pair.

```json
{
  "aggregators": ["jupiter", "okx"],
  "filter": { "pairs": [["So11111111111111111111111111111111111111112", "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"]] },
  "requote": true,
  "circularArbs": false,
  "detectFailedL1Swaps": false,
  "directFill": [],
  "extraMarkets": []
}
```

| Field | Default | Meaning |
| --- | --- | --- |
| `aggregators` | `["jupiter"]` | Whose original swaps to reroute: `jupiter`, `okx`, `titan`, `dflow`. |
| `filter` | every pair | Reroute only swaps that trade one of these mint pairs, in either direction. With `circularArbs`, an atomic arb is admitted when a listed pair names its mint. An empty `pairs` is rejected. |
| `requote` | `true` | Reroute via an aggregator, through the router. Set `false` to reroute directly to a venue only. |
| `circularArbs` | `false` | Also reroute atomic arbitrage. The router quotes each arb leg by leg; an arb that can't be split into legs fails with a `cycle_*` category. |
| `detectFailedL1Swaps` | `false` | Also reroute original swaps that failed onchain. |
| `directFill` | `[]` | Venue templates for rerouting directly to a venue. See [Rerouting Directly to a Venue](../../../flows/rerouting-historical-flow/rerouting-directly-to-a-venue.md). |
| `extraMarkets` | `[]` | Pools for the router to consider on top of the ones it knows. Needs `requote`. |

### continue

Executes the next blocks. Every field is optional, so `{}` executes one block.

```json
{
  "method": "continue",
  "params": {
    "advanceCount": 1,
    "transactions": ["<base64 transaction>"],
    "modifyAccountStates": { "<address>": { "data": { "data": "<base64>", "encoding": "base64" }, "executable": false, "lamports": 1000000, "owner": "<owner>", "space": 80 } }
  }
}
```

| Field | Default | Meaning |
| --- | --- | --- |
| `advanceCount` | `1` | Blocks to execute before the session waits again. A skipped slot has no block and doesn't count. |
| `transactions` | `[]` | User transactions. They run in order and commit before the first block. A failed user transaction doesn't produce an error; check it over the session's RPC endpoint. |
| `modifyAccountStates` | `{}` | Account overrides, written after `transactions` and before the first block. |

Wait for `readyForContinue` before sending the next `continue`. The server queues a `continue` sent early and runs it after the current one.

### continueTo

Runs to a slot, or to just before a batch in it, then pauses.

```json
{ "method": "continueTo", "params": { "slot": 380000050, "batchIndex": 3 } }
```

* `slot`: the slot to stop in. It must be after the last executed slot, or the server sends an `invalidRequest` error and the session stays where it is. It must have a block: the session doesn't stop at a skipped slot, and runs on to the end of the range.
* `batchIndex` (optional): stop before this batch executes. A batch is a group of the slot's transactions that execute together. While paused, RPC reads see the state up to batch `batchIndex - 1`. Without it the session runs the whole slot.

The session replies `paused`, then `readyForContinue`. Answer only the `readyForContinue`: answering both sends two requests.

### closeBacktestSession

Ends the session. The server replies `success` and closes the connection. In a parallel run, it ends every session and closes the connection without a reply.

```json
{ "method": "closeBacktestSession" }
```

### attachBacktestSession

Reattaches to a session after the connection dropped. Send it as the first message of a new connection, with the same API key.

```json
{ "method": "attachBacktestSession", "params": { "session_id": "backtest_…", "last_sequence": 42 } }
```

* `session_id`: from `sessionCreated`.
* `last_sequence` (optional): the last `seqId` you received. Omit it to replay every response the server still holds.

The server replies `sessionAttached`, then replays the responses you missed. Send `resumeAttachedSession` right after `sessionAttached`. See [Reattach](#reattach).

### resumeAttachedSession

Tells the server you are ready after reattaching. Send it right after `sessionAttached`. A session the server paused for a restart stays paused until you do; for any other session it does nothing. The server replies `success`, without a `seqId`.

```json
{ "method": "resumeAttachedSession" }
```

### Parallel Run Requests

A parallel run takes these instead of `continue`, `continueTo` and `attachBacktestSession`. The plain forms get an `invalidRequest` error.

```json
{ "method": "continueSessionV1", "params": { "sessionId": "backtest_…", "request": { "advanceCount": 1 } } }
{ "method": "continueToSessionV1", "params": { "sessionId": "backtest_…", "request": { "slot": 380000050 } } }
{ "method": "closeSessionV1", "params": { "sessionId": "backtest_…" } }
{ "method": "attachParallelControlSessionV2", "params": { "control_session_id": "parallel_…", "last_sequences": { "backtest_…": 17 } } }
```

* `request` takes the same fields as `continue` or `continueTo`.
* `closeSessionV1` ends one session of the parallel run. It sends no reply; an unknown `sessionId` gets `sessionNotFound`. To end the whole run, send `closeBacktestSession`.
* `attachParallelControlSessionV2` reattaches to the parallel run. `last_sequences` maps each session to the last `seq_id` you received from it; a session you leave out replays everything. No `resumeAttachedSession` is needed.

## Responses

| Method | `params` | Meaning |
| --- | --- | --- |
| `sessionCreated` | `{"session_id", "rpc_endpoint", "task_id"?, "session_name"?}` | The session exists. `rpc_endpoint` is a path such as `/backtest/<session_id>` on the server's host. Quote `task_id` when you report a problem. In a parallel run, sent once per session. |
| `sessionAttached` | as `sessionCreated` | The reply to `attachBacktestSession`. Missed responses follow. |
| `sessionsCreatedV2` | `{"control_session_id", "control_session_name"?, "session_ids", "task_ids", "start_slots", "end_slots"}` | Every session of a parallel run exists. Entry `i` of each list describes the same session. |
| `parallelSessionAttachedV2` | `{"control_session_id", "session_ids", "task_ids"}` | The reply to `attachParallelControlSessionV2`. |
| `readyForContinue` | none | The session is waiting for your next `continue` or `continueTo`. Read and change state over the session's RPC endpoint now. |
| `slotNotification` | a bare integer | The slot that just executed, as `getSlot` reports it. |
| `paused` | `{"slot", "batchIndex"?}` | The session stopped where a `continueTo` asked. |
| `discoveryBatch` | `{"slot", "batchIndex", "matched", "transactions", "transactionIndices"}` | An upcoming batch matches a discovery filter. `matched` lists the filters, `transactions` the matching transactions, `transactionIndices` their index in the block. |
| `status` | `{"status": "<value>"}` | Progress. See [Status Values](#status-values). |
| `success` | none | Acknowledges a request. The session also sends it after each executed block. |
| `completed` | `{"summary"?}` | The session executed its last slot. `summary` is present when you set `sendSummary`. |
| `error` | an error | See [Errors](#errors). |
| `sessionEventV2` | `{"session_id", "seq_id", "event"}` | An event from one session of a parallel run. `event` is `{"method", "params"?}` with any of `readyForContinue`, `slotNotification`, `paused`, `discoveryBatch`, `error`, `success`, `completed` and `status`. |

The session summary counts how the replayed transactions compare with what happened onchain:

```json
{
  "correctSimulation": 1520,
  "incorrectSimulation": 2,
  "executionErrors": 0,
  "balanceDiff": 1,
  "logDiff": 4,
  "unsanitizable": 0,
  "rerouteStats": { "swapsDetected": 310, "swapsRerouted": 280, "swapsSimulated": 280, "swapsSucceeded": 266 }
}
```

`rerouteStats` is present for a session with `reroute`. Its full field list is in [`RerouteStatsReport`](https://docs.rs/simulator-api/latest/simulator_api/struct.RerouteStatsReport.html).

## Status Values

`status` updates are informational; you don't need to act on them. A `continue` reports them in this order:

| Value | When |
| --- | --- |
| `startingRuntime` | While the session starts, about every 10 seconds, before `sessionCreated`. |
| `decodedTransactions` | The `continue`'s user transactions decoded. Only when it has some. |
| `executedUserTransactions` | The `continue`'s user transactions ran. Only when it has some. |
| `appliedAccountModifications` | The `continue`'s account overrides are written. |
| `readyToExecuteUserTransactions` | The `continue`'s inputs are applied. Blocks run next. |
| `executingBlockTransactions` | A block started executing. |
| `executedBlockTransactions` | A block finished executing. |
| `programAccountsLoaded` | Not sent by the current server. Accept it and ignore it. |

## Errors

An `error` names its kind in camelCase. A kind with fields is an object, with snake_case fields; a kind without fields is a bare string:

```json
{ "method": "error", "params": { "bundleNotFound": { "start_slot": 380000000, "end_slot": 380000099 } } }
{ "method": "error", "params": "noMoreBlocks" }
```

A terminal error ends the session. Any error that answers a create or an attach ends that request: no session was created or attached.

| Error | Fields | Terminal | Cause and fix |
| --- | --- | --- | --- |
| `invalidRequest` | `error` | No | The request is malformed, not allowed now, or asks for something the server can't serve, such as a range that isn't available. `error` says what to change. |
| `invalidTransactionEncoding` | `index`, `error` | No | A user transaction in a `continue` isn't valid base64. None of that `continue` ran: fix it and resend. |
| `invalidTransactionFormat` | `index`, `error` | No | A user transaction decodes but isn't a transaction. None of that `continue` ran. |
| `invalidAccountEncoding` | `address`, `encoding`, `error` | No | An account override's `data` doesn't decode. The user transactions ran; no override was written and no block executed. |
| `sessionNotFound` | `session_id` | No | The session you named doesn't exist or has ended. Stop: start a new session. |
| `sessionOwnerMismatch` | none | No | You attached with a different API key than the one that created the session. |
| `sessionOwnershipBusy` | `reason` | No | The session is moving between servers, or another attach raced yours. Retry the attach shortly. Also sent when you attach one session of a parallel run: attach the run instead. |
| `sessionNameTaken` | `session_name` | No | Names are never reused. Pick another, or resend the original create with its `createToken`. |
| `clerkError` | `error` | No | The server couldn't load historical data. During a create, it fails the create: retry later. |
| `simulationError` | `error` | No | An operation inside the session failed, such as reading an account. The session keeps running. |
| `noMoreBlocks` | none | Yes | The session has no block left to execute. |
| `bundleNotFound` | `start_slot`, `end_slot` | Yes | No bundle covers the range. Pick a range from `GET /available-ranges`, or build one with `POST /build-bundle`. |
| `noProducedBlocksInRange` | `start_slot`, `end_slot` | Yes | Every slot in the range was skipped onchain. Pick another range. |
| `advanceSlotFailed` | `slot`, `error` | Yes | The session couldn't start a slot. Report it with the `task_id`. |
| `finalizeSlotFailed` | `slot`, `error` | Yes | The session couldn't finish a slot. Report it with the `task_id`. |
| `internal` | `error` | Yes | The server failed. Retry later, and report it if it persists. |

`invalidAccountOwner`, `invalidAccountPubkey`, `invalidBlockhashFormat` and `initializingSysvarsFailed` exist in the protocol but the current server doesn't send them.

After a non-terminal error during a session, the session sends `readyForContinue` and waits for your next request.

## Reattach

Unless `disconnectTimeoutSecs` is `0`, a dropped connection doesn't end the session. You have that many seconds (30 by default) to reattach.

While detached, the session finishes the `continue` it was running, then waits. It never executes a block without a `continue`.

1. Open a new connection with the same API key, and send `attachBacktestSession` with the `session_id` and your last `seqId` as `last_sequence`.
2. Read `sessionAttached`, then send `resumeAttachedSession` right away.
3. Handle the responses that follow as usual. The replayed ones carry `seqId`s after your `last_sequence`, and can arrive before or after the `success` that answers the resume, which has no `seqId`.
4. Resubscribe on the session's RPC endpoint before your next `continue`. Pass `replayFromSlot` to get what you missed.

A `continue` you sent just before the drop ran if any response with a higher `seqId` than the `readyForContinue` it answered arrives, replayed or live: don't send it again, since the server queues every `continue` and would run it twice. If none arrives, the server never received it, and the session is waiting: send it again. Every `continue` the server receives answers with a `status` or an `error` before it executes a block.

When the server restarts, it closes the connection with close code `1012`. The session is paused, not ended: reattach as above.

If `disconnectTimeoutSecs` runs out first, the session ends and a later attach gets `sessionNotFound`.

The Rust client's managed session reconnects this way:

```
on a dropped connection, up to 20 attempts within 5 minutes, with backoff:
    connect with X-API-Key
    send attachBacktestSession { session_id, last_sequence }
    read responses until sessionAttached, handling each as usual
    send resumeAttachedSession
    read responses until success, handling each as usual
then resubscribe and carry on
```

After `completed`, the session waits for you to close it. Send `closeBacktestSession`, or close the connection.

## Parallel Sessions

`"parallel": true` on `createBacktestSession` splits the range into one session per bundle and runs them at once. The range may start inside a bundle; the first session then starts at that bundle's start.

1. The server sends `sessionCreated` for each session, then `sessionsCreatedV2` with the parallel run's `control_session_id` and each session's range.
2. Each session's events arrive as `sessionEventV2`, with that session's own `seq_id`. Keep the last one per session.
3. Drive each session with `continueSessionV1` or `continueToSessionV1`, and end one with `closeSessionV1`.
4. To reattach, send `attachParallelControlSessionV2` with your last `seq_id` per session.

`disconnectTimeoutSecs` defaults to 900 for a parallel run.
