# Quickstart

{% stepper %}
{% step %}
#### Create a Session

Start a new simulation session over a slot range.

* `startSlot` / `endSlot`: simulation range (inclusive).
* `accountEvents` (optional): stream updates for the listed accounts.
* `signerFilter` (optional): skip historical transactions signed by these addresses.
* `preloadPrograms` (optional): preload program accounts before execution.

This method requires an `X-API-Key` to be passed in the header.

{% code title="// via websocket to `ws(s)://<host>/backtest`" %}
```json
{
  "method": "createBacktestSession",
  "params": {
    "startSlot": 123456,
    "endSlot": 234567,
    "accountEvents": ["addr..."],
    "signerFilter": ["addr..."],
    "preloadPrograms": ["addr..."],
  }
}
```
{% endcode %}

The server also emits an initial `slotNotification` for `startSlot`, followed by `readyForContinue` when the session is ready.

```json
{
  "method": "sessionCreated",
  "params": {
    "sessionId": "<session_id>",
    "rpcEndpoint": "/backtest/<session_id>"
  }
}
```
{% endstep %}

{% step %}
#### Send Transactions

There are two supported paths for custom transactions:

* **RPC path:**
  * Send Solana transactions via `sendTransaction` to `http(s)://<host>/backtest/<session_id>`.
  * These execute immediately in the current slot after the historical block transactions.
* **WebSocket path:**
  * Include base64-encoded transactions in the `continue` request.
  * These execute in a batch, and the simulation also advances to the next slot.
  * See the section below.

Execution order per slot:

1. Historical block transactions.
2. User transactions submitted via RPC `sendTransaction`.
3. User transactions submitted in the `continue` batch.

> See [API Reference](api-reference.md#api-table) for the full list of supported Solana RPC methods.
{% endstep %}

{% step %}
#### Advance Slots

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

During a `continue`, the server emits `status` updates (e.g. `decodedTransactions`, `appliedAccountModifications`, `executedBlockTransactions`, `programAccountsLoaded`), `slotNotification` events as slots advance, and `readyForContinue` when it is safe to send the next request.
{% endstep %}

{% step %}
#### Close a Session

{% code title="// via websocket to `/backtest/<session_id>`" %}
```json
{
  "method": "closeBacktestSession"
}
```
{% endcode %}

This returns a `success` response and removes the session.
{% endstep %}

{% step %}
#### Account Updates

Any account registered in `accountEvents` streams real-time updates during execution.

```json
{
  "method": "accountNotification",
  "params": {
    "pubkey": "addr...",
    "result": {
      "context": { "slot": 123457 },
      "value": {
        "data": {
          "data": "<base64_bytes>",
          "encoding": "base64"
        },
        "executable": false,
        "lamports": 33594,
        "owner": "11111111111111111111111111111111",
        "rentEpoch": 100,
        "space": 80
      }
    }
  }
}
```

`result` is `null` if the account was removed.
{% endstep %}

{% step %}
#### Subscriptions

The per-session WebSocket (`/backtest/<session_id>`) supports standard Solana-style subscriptions, including:

* `accountSubscribe`
* `programSubscribe`
* `signatureSubscribe`
* `slotSubscribe`
* `logsSubscribe`

> See [API Reference](api-reference.md#api-table) for the full list of supported Solana subscription methods.
{% endstep %}
{% endstepper %}

#### Notes for Strategy Teams

* Use `preloadPrograms` + `status: programAccountsLoaded` if you need deterministic startup before your first `continue`.
* If you need rapid iteration, use `advanceCount > 1` to fast-forward while still receiving slot notifications.
* Signature checks are explicitly disabled, so transactions originating from any account can be spoofed.
