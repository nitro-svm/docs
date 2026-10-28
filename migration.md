# Migration

Upgrade to `simulator-client` 0.29, `simulator-api` 0.29 and `sim` 0.22. Find your current version, then work through each section from there down to the latest. Each section lists what stops compiling or changes behaviour, with the fix.

| You're on | Read |
| --- | --- |
| 0.15 | [All Versions](#all-versions), then every section below |
| 0.16 to 0.22 | [All Versions](#all-versions), then [From 0.16](#from-0-16) and the sections after it |
| 0.23 to 0.27 | [All Versions](#all-versions), then [From 0.23](#from-0-23) and [From 0.28](#from-0-28) |
| 0.28 | [All Versions](#all-versions), then [From 0.28](#from-0-28) |
| `sim` 0.21 or older | [Upgrading the CLI](#upgrading-the-cli) |
| Your own WebSocket client | [Raw WebSocket Clients](#raw-websocket-clients) |

## All Versions

1. Bump the client. It re-exports `simulator-api` as `simulator_client::simulator_api`, so you can drop your own `simulator-api` dependency. If you keep it, pin the same version.

   ```toml
   [dependencies]
   simulator-client = "0.29"
   ```
2. Use Rust 1.97 or newer.
3. Move your Solana crates to the versions in [Install](overview/installation/rust-reference.md#install). The 4.x Solana crates must be 4.3 or newer: the client moved from 4.1 (0.15, 0.16) to 4.2 (0.23) to 4.3 (0.28). `cargo tree -i solana-address@2 -e normal` shows which crates use the client's version.
4. Install a rustls crypto provider at the top of `main`, if you don't already. See [TLS](overview/installation/rust-reference.md#tls).

## From 0.15 <a href="#from-0-15" id="from-0-15"></a>

### `start_with_cancel` Is Now `start_with_params`

Before:

```rust
let session =
    ManagedBacktestSession::start_with_cancel(url, api_key, create, cancel.clone(), None).await?;
```

After:

```rust
use simulator_client::managed::StartSessionParams;

let params = StartSessionParams::builder()
    .url(url)
    .api_key(api_key)
    .create(create)
    .parent_cancel(cancel.clone())
    .build();
let session = ManagedBacktestSession::start_with_params(params, None).await?;
```

Without a cancellation token, call `ManagedBacktestSession::start(url, api_key, create)` as before.

### Managed Subscriptions Take Addresses

`subscribe_transactions` and `subscribe_account_diffs` on `ManagedBacktestSession` take `Vec<Address>` instead of `Vec<String>`.

Before:

```rust
session.subscribe_transactions(vec![program_id.to_string()]);
```

After:

```rust
use simulator_client::Address;

let program_id: Address = "<PROGRAM_ID>".parse()?;
session.subscribe_transactions(vec![program_id]);
```

## From 0.16 <a href="#from-0-16" id="from-0-16"></a>

These apply to 0.16 through 0.22.

### Reroutes Arrive as Replacements

`subscribe_reroutes` and `RerouteNotification` are gone. `subscribe_replacements` streams every reroute result as a `ReplacementNotification`: one kind for a reroute via an aggregator, one for a reroute directly to a venue, and the original swap as the baseline.

Before:

```rust
let handle = subscribe_reroutes(&rpc_endpoint, |n: RerouteNotification| async move {
    println!("{}", n.original_signature);
})
.await?;
```

After:

```rust
use simulator_client::{ReplacementNotification, subscribe_replacements};

let handle = subscribe_replacements(&rpc_endpoint, |n| async move {
    match n {
        ReplacementNotification::Requote(requote) => { /* via an aggregator */ }
        ReplacementNotification::DirectFill(probe) => { /* directly to a venue */ }
        ReplacementNotification::Original(original) => { /* the original swap */ }
    }
})
.await?;
```

On a managed session, call `session.subscribe_replacements()` instead and handle `ManagedEvent::Replacement` in your event loop. Every kind carries the shared fields in `n.core()`. See [Rerouting Historical Flow](flows/rerouting-historical-flow/README.md).

### The Arbitrage Agent Is Gone

The built-in arbitrage agent was removed, with no replacement. Delete `CreateSession::agents` and calls to `report_agent_pnl`.

### Bare Hosts Work

`ManagedBacktestSession::start` and `BacktestClient::builder().url(..)` accept a bare host such as `simulator.termina.technology`. A full `wss://…/backtest` URL still works.

## From 0.23 <a href="#from-0-23" id="from-0-23"></a>

These apply to 0.23 through 0.27.

### The Signer Filter Is Gone

`CreateSession::signer_filter` and `add_signer_filter` were removed, with no replacement. Delete them.

Before:

```rust
let create = CreateSession::builder()
    .start_slot(start)
    .slot_count(100)
    .build()
    .add_signer_filter(signer)
    .into_request()?;
```

After:

```rust
let create = CreateSession::builder()
    .start_slot(start)
    .slot_count(100)
    .build()
    .into_request()?;
```

### Scheduled Actions Have a `resolve` Field

A `ScheduledAction` struct literal needs the new `resolve` field. `None` keeps the old behaviour.

```rust
ScheduledAction {
    anchor: ActionAnchor::AfterSlot { slots },
    kind: ActionKind::Simulate,
    transactions,
    account_overrides,
    feeds_reroute: false,
    return_accounts,
    label: Some("my-action".into()),
    resolve: None, // new
}
```

### `ManagedEvent` Has a `Replacement` Variant

From 0.26, a `match` on `ManagedEvent` without a `_` arm needs a `ManagedEvent::Replacement(_)` arm.

## From 0.28 <a href="#from-0-28" id="from-0-28"></a>

### New Error Variants

Add arms for these, or a `_` arm, wherever you `match` on them exhaustively:

| Enum | New variants |
| --- | --- |
| `ManagedSessionError` | `ApiKeyRejected`, `Remote`, `CompletedBeforeReady` |
| `SubscriptionRuntimeError` | `Task` |

What each one means is in [Errors](overview/installation/rust-reference.md#errors).

### A Bad API Key Fails at Once

`ManagedBacktestSession::start` used to retry a rejected API key for up to 5 minutes, which looked like a hang. It now returns `ManagedSessionError::ApiKeyRejected` on the first HTTP 401 or 403. `sim` prints `the server rejected the API key` and exits with status 1.

### Optional: Shorter Code

None of these is required, but each replaces code you likely wrote yourself.

**Drive a session to completion.** Before:

```rust
loop {
    match session.next_event().await? {
        ManagedEvent::ReadyForContinue | ManagedEvent::Paused(_) => {
            session.send_continue(step.clone()).await?;
        }
        ManagedEvent::Completed { .. } => break,
        ManagedEvent::Error(err) if err.is_terminal() => return Err(err.into()),
        _ => {}
    }
}
session.shutdown().await;
```

After:

```rust
let summary = session
    .run_to_completion(step, |event| {
        if let ManagedEvent::Slot(slot) = event {
            println!("executed slot {slot}");
        }
    })
    .await?;
```

`run_to_completion` stops on any `ManagedEvent::Error`, not only terminal ones, and shuts the session down before it returns. To read state before the first step, call `session.wait_until_ready().await?`, then send the first `send_continue` yourself.

**Read state through the session's RPC endpoint.** Before:

```rust
let rpc = RpcClient::new(session.session_info().rpc_endpoint.clone());
let account = rpc.get_account(&address).await?;
```

After:

```rust
let account = session.rpc().get_account(&address).await?;
```

**Stop a subscription.** `shutdown` drains what is buffered, waits for your callbacks and returns their errors. Before:

```rust
handle.stop.send(true).ok();
handle.join_handle.await.ok();
```

After:

```rust
handle.shutdown().await?;
```

**Encode and decode transactions.** For `ScheduledAction::transactions` or `ContinueParams::transactions`, use `encode_transaction` instead of your own base64 and bincode; for `DiscoveryBatchEvent::transactions`, `ActionResultNotification::matched` or `RequoteNotification::routed_transaction`, use `decode_transaction` (or `ActionResultNotification::matched_transaction`). Bincode drops a version-1 transaction's version marker, so wincode is used on both sides. Before:

```rust
let encoded = BASE64_STANDARD.encode(bincode::serialize(&tx)?);
```

After:

```rust
let encoded = simulator_client::encode_transaction(&tx)?;
let decoded = simulator_client::decode_transaction(&binary)?;
```

**Import addresses and protocol types from the client.** Use `simulator_client::Address` instead of `solana_address::Address`, and `simulator_client::simulator_api::…` instead of `simulator_api::…`, so their versions always match the client's.

## Upgrading the CLI

Pin `sim` 0.22 with `cargo install sim-cli --locked --version 0.22.0`, or get the latest build with `sim update`. Check what you run with `sim --version`.

* **`--url` goes after the command.** Only commands that reach the server take it; `compare`, `summarize` and `update` don't. `SIMULATOR_URL` works as before.

  ```bash
  # Before
  sim --url staging.simulator.termina.technology run --start-slot <START> --end-slot <END>
  # After
  sim run --url staging.simulator.termina.technology --start-slot <START> --end-slot <END>
  ```
* **`sim compare --fail-on-diff`** is new. Before, `sim compare` always exited 0, so a CI job that relied on its status always passed. Add the flag to fail on differences. See [Use in CI](cli/README.md#use-in-ci).
* **A rejected API key fails at once**, with status 1, instead of retrying for up to 5 minutes.
* **`sim bundle build --latest` builds exactly `--bundle-size` slots.** Before, it built one slot more.

## Raw WebSocket Clients

If you send your own JSON, check these wire changes. The server ignores fields it doesn't know, so a removed field fails silently: your request runs, without the setting.

| Before | Now | Changed in |
| --- | --- | --- |
| `"rerouteOrderFlow": true` | `"reroute": {}`. Options such as `aggregators`, `requote` and `directFill` go inside the object. See [Reroute](overview/installation/api-reference/session-control-channel.md#reroute). | 0.23 |
| `"agents": [...]` | Removed. | 0.23 |
| `rerouteSubscribe` / `rerouteNotification` | `replacementSubscribe` / `replacementNotification`, tagged by `kind`. See [replacementNotification](overview/installation/api-reference/session-rpc.md#replacementnotification). | 0.23 |
| `"signerFilter": [...]` | Removed. | 0.28 |
| `attachBacktestSession` with `sessionId` and `lastSequence` | `session_id` and `last_sequence`. Fields inside a request variant are snake_case. See [Field Casing](overview/installation/api-reference/README.md#field-casing). | Earlier docs were wrong; the wire didn't change. |

The full protocol is in the [API Reference](overview/installation/api-reference/README.md).
