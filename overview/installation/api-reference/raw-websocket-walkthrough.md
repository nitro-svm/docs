# Raw WebSocket Walkthrough

One session from create to close over a plain WebSocket, then a reattach. In the transcripts, `>` is a message you send and `<` one the server sends. `seqId` values are illustrative.

{% stepper %}
{% step %}
#### Pick a Slot Range

```bash
curl -s -H "X-API-Key: $SIMULATOR_API_KEY" https://simulator.termina.technology/available-ranges
```

Take a range's `bundle_start_slot` as `startSlot`, and an `endSlot` at or before its `max_bundle_end_slot`.
{% endstep %}

{% step %}
#### Create the Session

Connect to `wss://simulator.termina.technology/backtest` with the header `X-API-Key: <API_KEY>`, and send `createBacktestSession` as the first message.

```
> {"method":"createBacktestSession","params":{"startSlot":380000000,"endSlot":380000001,"sendSummary":true,"disconnectTimeoutSecs":300}}
< {"method":"status","params":{"status":"startingRuntime"}}
< {"method":"sessionCreated","params":{"session_id":"backtest_019a2b3c4d5e6f708192a3b4c5d6e7f8","rpc_endpoint":"/backtest/backtest_019a2b3c4d5e6f708192a3b4c5d6e7f8","task_id":"…","session_name":"…"}}
< {"seqId":0,"method":"readyForContinue"}
< {"seqId":1,"method":"slotNotification","params":379999999}
```

* Keep `session_id`: you need it to reattach.
* The session starts at the slot before `startSlot`, and is ready for RPC reads and your first `continue`.
* Keep the last `seqId` you received.
{% endstep %}

{% step %}
#### Read State

Call the session's RPC endpoint over HTTP: the server's host plus `rpc_endpoint`.

```bash
curl -s https://simulator.termina.technology/backtest/backtest_019a2b3c4d5e6f708192a3b4c5d6e7f8 \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getSlot"}'
```

```json
{"jsonrpc":"2.0","id":1,"result":379999999}
```
{% endstep %}

{% step %}
#### Execute Blocks

Send `continue` on the session control channel. Wait for `readyForContinue` before the next one.

```
> {"method":"continue","params":{"advanceCount":1}}
< {"seqId":2,"method":"status","params":{"status":"appliedAccountModifications"}}
< {"seqId":3,"method":"status","params":{"status":"readyToExecuteUserTransactions"}}
< {"seqId":4,"method":"status","params":{"status":"executingBlockTransactions"}}
< {"seqId":5,"method":"status","params":{"status":"executedBlockTransactions"}}
< {"seqId":6,"method":"slotNotification","params":380000000}
< {"seqId":7,"method":"success"}
< {"seqId":8,"method":"readyForContinue"}
```

To add user transactions or account overrides, put them in `transactions` and `modifyAccountStates`; see [continue](session-control-channel.md#continue).
{% endstep %}

{% step %}
#### Finish and Close

The last block ends with `completed` instead of `readyForContinue`. Then close the session.

```
> {"method":"continue","params":{"advanceCount":1}}
< {"seqId":9,"method":"status","params":{"status":"appliedAccountModifications"}}
< …
< {"seqId":14,"method":"slotNotification","params":380000001}
< {"seqId":15,"method":"success"}
< {"seqId":16,"method":"completed","params":{"summary":{"correctSimulation":2875,"incorrectSimulation":0,"executionErrors":0,"balanceDiff":0,"logDiff":1,"unsanitizable":0}}}
> {"method":"closeBacktestSession"}
< {"method":"success"}
```
{% endstep %}
{% endstepper %}

## Reattach

Say the connection dropped after `seqId` 8. Within `disconnectTimeoutSecs`, open a new connection to `wss://simulator.termina.technology/backtest` with the same API key:

```
> {"method":"attachBacktestSession","params":{"session_id":"backtest_019a2b3c4d5e6f708192a3b4c5d6e7f8","last_sequence":8}}
< {"method":"sessionAttached","params":{"session_id":"backtest_019a2b3c4d5e6f708192a3b4c5d6e7f8","rpc_endpoint":"/backtest/backtest_019a2b3c4d5e6f708192a3b4c5d6e7f8","task_id":"…","session_name":"…"}}
< {"seqId":9,"method":"…"}
```

1. Send `resumeAttachedSession` right after `sessionAttached`:

   ```
   > {"method":"resumeAttachedSession"}
   < {"method":"success"}
   ```
2. Handle the replayed responses, everything after `last_sequence`, as usual. They carry a `seqId` and can arrive before or after the `success`, which has none.
3. Resubscribe on the session's RPC endpoint, if you had subscriptions.

Then carry on with `continue` as before. The [Session Control Channel](session-control-channel.md#reattach) page covers a `continue` that was in flight, close code `1012` and the errors an attach can get.
