# API Reference

#### Architecture

* **Historical State Archive**
  * Accounts, blocks, and deltas are stored in Parquet for fast deterministic queries.
* **Replay Environment**
  * A Solana-compatible execution layer that replays historical transactions and applies injected ones.
* **Session Control (WebSocket)**
  * Endpoint: `ws(s)://<host>/backtest`
  * Methods: `createBacktestSession`, `continue`, `closeBacktestSession`
  * Response events: `sessionCreated`, `readyForContinue`, `slotNotification`, `accountNotification`, `status`, `completed`, `error`
* **Per-Session RPC (HTTP + WebSocket)**
  * Endpoint: `http(s)://<host>/backtest/{session_id}`
  * WebSocket subscriptions: `ws(s)://<host>/backtest/{session_id}`
  * Supported JSON-RPC methods include:
    * `getAccountInfo`, `getBalance`, `getMultipleAccounts`, `getProgramAccounts`
    * `getLatestBlockhash`, `getFeeForMessage`, `getSignatureStatuses`, `getTransaction`
    * `simulateTransaction`, `sendTransaction`, `getSlot`, `getAddressLookupTable`
    * `modifyAccounts`

> Note: `/backtest` is authenticated by API key in production. `/backtest/{session_id}` is unauthenticated for per-session access.

#### API Table

| Method                  | Purpose                                                                             | Surface                        |
| ----------------------- | ----------------------------------------------------------------------------------- | ------------------------------ |
| `createBacktestSession` | Start a backtest session and receive `sessionId` + `rpcEndpoint`.                   | Session control (WS)           |
| `continue`              | Execute a batch and advance slots; supports `transactions` + `modifyAccountStates`. | Session control (WS)           |
| `closeBacktestSession`  | Close and clean up a session.                                                       | Session control (WS)           |
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
