# Troubleshooting

Find the message you see, then apply the fix. Messages are shown as the CLI (`sim` 0.22) and the Rust client (`simulator-client` 0.29) print them; `...` stands for the parts that vary.

## Turn On Logs

Start here when something hangs or fails without a clear message.

* **CLI:** set `RUST_LOG`, which works for every command. Logs go to stderr. With `sim run`, also add `--verbose`, which swaps the progress bar for logs.

  ```bash
  RUST_LOG=info sim run --verbose --start-slot <START> --end-slot <END> --program-id <PROGRAM_ID>
  ```
* **Rust:** the client logs through `tracing`, so it prints nothing until you install a subscriber. Add `tracing-subscriber = { version = "0.3", features = ["env-filter"] }`, call this at the top of `main`, and run with `RUST_LOG=simulator_client=info` (`debug` also shows reattaches):

  ```rust
  tracing_subscriber::fmt()
      .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
      .init();
  ```

## API Key

| Message | Cause | Fix |
| --- | --- | --- |
| `the following required arguments were not provided: --api-key <API_KEY>` (exit status 2) | No key is set. `sim` stops before reaching the server. | Set `SIMULATOR_API_KEY`, or pass `--api-key`. |
| `the server rejected the API key (HTTP 401 Unauthorized); check SIMULATOR_API_KEY or --api-key` (`sim run`) | The key is mistyped, or no longer valid. | Check `SIMULATOR_API_KEY` or `--api-key`. Run `sim usage` to check the key. |
| `HTTP 401 Unauthorized from ...` from `sim ranges`, `sim usage`, `sim bundle` or another command | Same. | Same. |
| `HTTP 403 Forbidden from .../generate-depth: generate_depth feature not enabled` (`sim depth`, `POST /generate-depth`) | The key works, but custom datasets aren't enabled on it. Only depth requests return `403`. | [Contact us](README.md#get-connected) to enable custom datasets. |
| `API key rejected: HTTP 401 Unauthorized` (`ManagedSessionError::ApiKeyRejected`) | The key is mistyped, or no longer valid. The client never retries it. | Fix the key you pass to `ManagedBacktestSession::start`. |
| `websocket connect to wss://.../backtest failed: HTTP error: 401 Unauthorized` (`BacktestClientError::Connect`) | Same, on the raw client. | Fix the key you pass to `BacktestClient::builder().api_key(..)`. |
| `HTTP 401 Unauthorized from ...` (`BacktestClientError::HttpStatus`), from `available_ranges` and the other HTTP calls | Same. | Same. |

`sim usage` is the quickest key check: it prints your key's usage, or fails with `401` and exits with status 1.

Keys don't expire on their own. A key that was revoked gets `401` everywhere, like a mistyped one. To get a new key, see [Get Connected](README.md#get-connected).

## Slot Ranges

| Message | Cause | Fix |
| --- | --- | --- |
| `session create failed: ... block bundle not found for start_slot=..., end_slot=...` | No bundle covers the range. | Run `sim ranges` (or call `available_ranges`) and pick a listed range. For an older range, build a custom bundle with `sim bundle build`. |
| `session create failed: ... invalid request: ...` | The request asks for something the server can't serve, such as a range that isn't available or a start slot that isn't a range's first slot. The rest of the message says what to change. | A single session must start exactly at a range's `bundle_start_slot` and end at or before its `max_bundle_end_slot`. Longer ranges: use `sim run --parallel`. |
| `session create failed: ... no produced blocks in requested range ...` | Every slot in the range was skipped onchain. | Widen or move the range. |
| `cannot split range: ...` or `unable to generate plan: gap in data between slots ... and ...` (`sim run --parallel`, `--plan`) | Part of the range has no bundle. | Pick a range that `sim ranges` lists without gaps, or build a custom bundle for the missing slots. |
| `no more blocks available` (`noMoreBlocks`) | You stepped past the session's last slot. | Stop: the session is complete. |

## Server Capacity

| Message | Cause | Fix |
| --- | --- | --- |
| `session create failed: ... unable to start session: insufficient capacity` | The server had no room for a new session within the wait. | **CLI:** `sim run` already waits up to 900 seconds; retry later. **Rust:** set `capacity_wait_timeout_secs` on `CreateSession`. Unset, the create fails at once when the server is busy. |
| `clerk error: ...` (`clerkError`) when creating a session | The server couldn't load the range's historical data. | Retry later. |
| `HTTP 429 Too Many Requests` from `sim bundle build`, `sim depth measure` or their endpoints | Your key used up its request limit for bundle builds and depth jobs. | Wait, then send the request again. See [Usage and Limits](cli/README.md#usage-and-limits). |

## Sessions and Reattach

A session runs on the server and outlives its connection, unless `disconnectTimeoutSecs` is `0`. `sim` and `ManagedBacktestSession` reattach on their own; the raw `BacktestClient` doesn't.

| Message | Cause | Fix |
| --- | --- | --- |
| `session not found: ...` (`sessionNotFound`) when you reattach | The session ended: it completed, or the reattach came after its disconnect window ran out (below). | Start a new session. On the raw client, set `disconnect_timeout_secs` on the create. |
| `control failed: ...` (`ManagedSessionError::ControlFailed`) | The managed client ran out of reconnect attempts (20 within 5 minutes), or the server ended the session. | Start a new session. |
| `session owner mismatch` (`sessionOwnerMismatch`) | You reattached with a different API key from the one that created the session. | Reattach with the key that created it. |
| `session ownership busy: ...` (`sessionOwnershipBusy`) | The session is moving to another server, or another attach raced yours. | Retry the attach after a second or two. A managed session does this for you. |
| `session name '...' is already in use` (`sessionNameTaken`) | Run and session names are never reused, even after the session ends. | Pick another `--name` (or `name`), or leave it unset to get a generated one. |
| `failed to advance to slot ...`, `failed to finalize slot ...` or `internal error: ...` | The server failed. These end the session. | Retry later. If it persists, report it with the `task_id` (see [Still Stuck](#still-stuck)). |

How long the server keeps a session after its connection drops (its disconnect window):

| Client | Window |
| --- | --- |
| `sim run`, with or without `--parallel` | 180 seconds |
| `ManagedBacktestSession` | 360 seconds |
| `ManagedParallelSession`, or a raw `parallel` create, with `disconnect_timeout_secs` unset | 900 seconds |
| Raw single session, `disconnect_timeout_secs` unset | 30 seconds |
| `disconnect_timeout_secs` set to `0` | None: the session ends with the connection |

`disconnect_timeout_secs` takes at most 900.

## Rust Client

| Symptom | Cause | Fix |
| --- | --- | --- |
| Panic: `Could not automatically determine the process-level CryptoProvider from Rustls crate features` | More than one rustls crypto provider is linked (rustls's default features, `reqwest` 0.13 or the AWS SDK pull in `aws-lc-rs`). | Install one at the top of `main`, before you connect: `let _ = rustls::crypto::ring::default_provider().install_default();` |
| Compile error: expected `Address`, found `Pubkey`, or a trait bound not satisfied on a Solana type | A Solana crate from a different major version than the client's. | Use the versions in [Install](overview/installation/rust-reference.md#install). `cargo tree -i solana-address@2 -e normal` shows which crates use the client's version. Use `simulator_client::Address`, or convert with `Address::new_from_array(pubkey.to_bytes())`. |
| `cargo tree` shows several versions of `solana-address` | The `spl-*` crates pull in older major versions (1.x, 3.x). | Nothing, if your code compiles: those copies are harmless. |
| `invalid transaction format at index ...` (`invalidTransactionFormat`) | A user transaction wasn't encoded the way the server reads it. | Encode each transaction with `simulator_client::encode_transaction`, or add it with `Continue::builder().build().push_transaction(&tx)?`. |
| `server error: ...` (`ManagedSessionError::Remote`) from `wait_until_ready` or `run_to_completion` | The server reported an error. These two methods stop on any error, even one that leaves the session running. | Fix what the message names, then start a new session. |
| `remote error: ...` (`BacktestClientError::Remote`) on the raw client | The server refused a request. The rest is the error from the [errors table](overview/installation/api-reference/session-control-channel.md#errors). | Fix what it names. If it is terminal, start a new session. |
| `session completed before it was ready to continue` (`CompletedBeforeReady`) | The session finished before its first pause. | Check the range and the create request. |
| Session RPC calls fail, or return HTTP 502 | The session's RPC endpoint serves only from the first `ReadyForContinue` until `Completed`. | Call `session.wait_until_ready().await?` before the first read, and read before the session completes. |
| `AccountNotFound: pubkey=...` from `session.rpc().get_account` | The account doesn't exist in the session's state. | Use `get_multiple_accounts`, which returns `None` for a missing account. |
| `subscription failed: ...` (`ManagedSessionError::SubscriptionFailed`) | A subscription ran out of retries, so its notifications are incomplete. | Rerun the session if you need every notification. |

`simulation error: ...` in the logs is not fatal: an operation inside the session failed, such as reading an account, and the session keeps running.

## CLI Results

| Symptom | Cause | Fix |
| --- | --- | --- |
| The output file has no `tx` lines | `sim run` records transactions only for the programs you name. | Add `--program-id <PROGRAM_ID>` (repeat it for more programs). |
| `sim compare` exits 0 although the runs differ | `sim compare` reports differences but succeeds unless you ask it to fail. | Add `--fail-on-diff`. Add the `regressions` section to fail only when a transaction went from success to failure. |
| `no matching transactions found in the requested slot range` (exit status 1) | The program you passed with `--program-id` ran no transaction in the range. | Check the program ID, or pick another range. |
| The output file has no `summary` line, `sim run` exited with status 130, or `sim summarize` warns that its totals are partial | The run was stopped with Ctrl+C, SIGTERM or SIGQUIT. The file holds what was recorded until then. | Rerun the range to get a complete file. `sim compare` rejects a file without a `summary` line. |
| `sim compare` fails with ``... has no `summary` line`` or `... share no transactions` | One run didn't finish, or the runs recorded different slots or programs. | Rerun both with the same range and `--program-id`. |
| `unexpected argument '--url' found` | Since `sim` 0.22, `--url` goes after the command, and only commands that reach the server take it. | Write `sim run --url <HOST>`, not `sim --url <HOST> run`. See [Migration](migration.md#upgrading-the-cli). |
| `--venue-templates is only read under --reroute-order-flow venue` (or the same for `--reroute-extra-markets` and `router`) | The flag belongs to the other reroute mode. | Pair `--venue-templates` with `--reroute-order-flow venue`, and `--reroute-extra-markets` with `--reroute-order-flow router`. |

For exit statuses, see [Exit Status](cli/README.md#exit-status). For file formats, see [File Formats](cli/file-formats.md).

## Still Stuck

[Contact us](README.md#get-connected) with:

* the full message, and the logs from [Turn On Logs](#turn-on-logs);
* the run or session name (printed as `Session name:` when `sim run` starts);
* the `session_id` and `task_id` from the `session_started` line of the output file, or from `session_info()` in Rust;
* the `sim --version` output, or your `simulator-client` version.
