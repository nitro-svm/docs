# API Reference

#### Architecture

* **Historical State Archive**
  * Accounts, blocks, and deltas are stored in a proprietary format for fast deterministic queries.
* **Available Ranges**
  * Before creating a session, confirm the slot range of interest is available.
  * `curl https://<host>/available-ranges | jq`&#x20;
* **Session Control Channel**
  * Connect to create and drive a backtest session. All messages on this channel use a custom JSON protocol.
  * Endpoint: `ws(s)://<host>/backtest`
  * Methods: `createBacktestSession`, `attachBacktestSession`, `continue`, `closeBacktestSession`
  * Responses: `sessionCreated`, `sessionAttached`,`readyForContinue`, `slotNotification`, `status` (see variants below), `success`, `completed`, `error`&#x20;
*   **Sequenced Responses**

    * When `disconnectTimeoutSecs > 0`, responses are wrapped with a sequence id so clients can resume a session after reconnecting.
    * `seqId` is a monotonically increasing sequence number. Clients can pass their last received sequence when calling `attachBacktestSession` to resume the stream without replaying earlier messages.

    ```json
    {
      "seqId": 123,
      "method": "status",
      "params": {
        "status": "decodedTransactions"
      }
    }
    ```
* **Per-Session RPC Channel**
  * Once a session is created, interact with the simulated Solana environment. Most methods conform to the standard Solana JSON-RPC interface.
  * Endpoint: `http(s)://` or `ws(s)://` at `/backtest/{session_id}`
  * Account state: `getAccountInfo`, `getBalance`, `getMultipleAccounts`, `getProgramAccounts`
  * Chain state: `getLatestBlockhash`, `getFeeForMessage`, `getSlot`&#x20;
  * Transactions: `simulateTransaction`, `sendTransaction`, `getTransaction`
  * Subscriptions: `accountSubscribe`, `slotSubscribe`, `transactionSubscribe`

> Notes:
>
> * &#x20;`/backtest` is authenticated by API key in production. `/backtest/{session_id}` is unauthenticated for per-session access.
> * `ReadyToExecuteUserTransactions` is the main status typically requires to wait for before sending or executing user transactions.
> * If your client script does strict parsing, make sure it can handle above mentioned status variants.

#### API Table

| Method                  | Purpose                                                                             | Surface                        |
| ----------------------- | ----------------------------------------------------------------------------------- | ------------------------------ |
| `createBacktestSession` | Start a backtest session and receive `sessionId` + `rpcEndpoint`.                   | Session control (WS)           |
| `continue`              | Execute a batch and advance slots; supports `transactions` + `modifyAccountStates`. | Session control (WS)           |
| `closeBacktestSession`  | Close and clean up a session.                                                       | Session control (WS)           |
| `attachBacktestSession` | Reattach to an existing session and receive rpcEndpoint.                            | Session control (WS)           |
| `getAccountInfo`        | Fetch a single account at current slot.                                             | Per-session RPC (HTTP)         |
| `getBalance`            | Fetch lamport balance for an account.                                               | Per-session RPC (HTTP)         |
| `getMultipleAccounts`   | Fetch multiple accounts in one call.                                                | Per-session RPC (HTTP)         |
| `getProgramAccounts`    | Fetch accounts owned by a program.                                                  | Per-session RPC (HTTP)         |
| `getLatestBlockhash`    | Get current slot blockhash.                                                         | Per-session RPC (HTTP)         |
| `getFeeForMessage`      | Get fee estimate for a message.                                                     | Per-session RPC (HTTP)         |
| `getSignatureStatuses`  | Fetch status for signatures.                                                        | Per-session RPC (HTTP)         |
| `getTransaction`        | Fetch transaction details (including logs).                                         | Per-session RPC (HTTP)         |
| `simulateTransaction`   | Simulate a transaction without committing state.                                    | Per-session RPC (HTTP)         |
| `sendTransaction`       | Execute a transaction immediately in the current slot.                              | Per-session RPC (HTTP)         |
| `getSlot`               | Get the current slot.                                                               | Per-session RPC (HTTP)         |
| `getAddressLookupTable` | Fetch an ALT account.                                                               | Per-session RPC (HTTP)         |
| `modifyAccounts`        | Apply account overrides via RPC.                                                    | Per-session RPC (HTTP)         |
| `accountSubscribe`      | Stream account changes.                                                             | Per-session subscriptions (WS) |
| `programSubscribe`      | Stream program-owned account changes.                                               | Per-session subscriptions (WS) |
| `signatureSubscribe`    | Stream signature status updates.                                                    | Per-session subscriptions (WS) |
| `slotSubscribe`         | Stream slot updates.                                                                | Per-session subscriptions (WS) |
| `logsSubscribe`         | Stream log updates.                                                                 | Per-session subscriptions (WS) |
