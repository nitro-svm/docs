# Quickstart

{% stepper %}
{% step %}
#### Create Session

Start a new simulation session over a slot range.

* `startSlot` / `endSlot`: simulation range (inclusive).
* `signerFilter` (optional): skip historical transactions signed by these addresses.
* `disconnectTimeoutSecs` (optional): keep the session alive after disconnect (max is 900s).

This method requires an `X-API-Key` to be passed in the header.

{% code title="// via websocket to `ws(s)://<host>/backtest`" %}
```json
{
  "method": "createBacktestSession",
  "params": {
    "startSlot": 123456,
    "endSlot": 234567,
    "signerFilter": ["addr..."],
    "disconnectTimeoutSecs": 30
  }
}
```
{% endcode %}

The server also emits an initial `slotNotification` for `startSlot`, followed by `readyForContinue` when the session is ready. Use the returned RPC endpoint for session-specific interactions.

```json
{
  "method": "sessionCreated",
  "params": {
    "sessionId": "<session_id>",
    "rpcEndpoint": "/backtest/<session_id>"
  }
}
```

> This WebSocket connection needs to stay open for the duration of the session. If it's closed, the session will be terminated and all account state removed.
{% endstep %}

{% step %}
#### Read Accounts

Each session supports standard Solana JSON-RPC methods and subscriptions at `/backtest/<session_id>`, including:

* `getAccountInfo`, `getBalance`, `getMultipleAccounts`
* `accountSubscribe`, `programSubscribe`, `signatureSubscribe`&#x20;

> See [API Reference](api-reference.md#api-table) for the full list of supported Solana subscription methods.
{% endstep %}

{% step %}
#### Send Transactions

There are two supported paths for custom transactions:

* **RPC path:**
  * Submit transactions via the standard Solana RPC method `sendTransaction`.
  * These execute immediately in the current slot after the historical block transactions.
* **WebSocket path:**
  * Include base64-encoded transactions in the `continue` request.
  * These execute in a batch and also trigger the simulation to advance to the next slot.

Execution order per slot:

1. Historical block transactions.
2. User transactions submitted via RPC `sendTransaction`.
3. User transactions included in the `continue` batch.
{% endstep %}

{% step %}
#### Execute Slots

The simulator starts at `startSlot` and waits for an explicit `continue` before proceeding to subsequent slots.

* `advanceCount`: number of slots to execute before pausing again for user action, defaults to `1` when omitted.
* `transactions`: custom transactions.
* `modifyAccountStates`: account overrides applied before execution.

{% code title="// via websocket to `/backtest/<session_id>`" %}
```json
{
  "method": "continue",
  "params": {
    "advanceCount": 1,
    "transactions": ["<base64_encoded_transaction>"],
    "modifyAccountStates": {
      "addr...": {
        "data": {
          "data": "<base64_bytes>",
          "encoding": "base64"
        },
        "executable": false,
        "lamports": 33594,
        "owner": "11111111111111111111111111111111",
        "space": 80
      }
    }
  }
}

```
{% endcode %}

During a `continue`, the server emits `status` updates (e.g. `preparingBundle`, `decodedTransactions`, `appliedAccountModifications`, `executedBlockTransactions`, `programAccountsLoaded`). It'll emit `readyForContinue` when it's ready to advance to the next slot.
{% endstep %}

{% step %}
#### Reattach Session <a href="#reattach-session" id="reattach-session"></a>

If the session connection drops by accident, reattach within the recovery window. WebSocket subscription events are buffered for the last 100 slots.

{% code title="// via websocket to `/backtest`" %}
```json
{
  "method": "attachBacktestSession",
  "params": {
    "sessionId": "<session_id>",
    "lastSequence": 42
  }
}
```
{% endcode %}

Note that `lastSequence` is optional. If omitted, the server replays the entire buffered stream.


{% endstep %}

{% step %}
#### Close Session

{% code title="// via websocket to `/backtest/<session_id>`" %}
```json
{
  "method": "closeBacktestSession"
}
```
{% endcode %}

This returns a `success` response and removes the session.
{% endstep %}
{% endstepper %}

#### Notes for Strategy Teams

* Use `preloadPrograms` + `status: programAccountsLoaded` if you need deterministic startup before your first `continue`.
* If you need rapid iteration, use `advanceCount > 1` to fast-forward while still receiving slot notifications.
* Signature checks are explicitly disabled, so transactions originating from any account can be spoofed.
