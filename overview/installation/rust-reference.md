# Rust Reference

The `simulator-client` crate is the Rust client for the Termina simulator. It has two layers:

* **Managed** (`ManagedBacktestSession`): use this one. It reconnects and reattaches on its own, so a dropped connection doesn't end your session.
* **Raw** (`BacktestClient` and `BacktestSession`): one WebSocket per session, no reconnect. Use it when you need full control, such as pausing on discovery filters.

`simulator-client` re-exports the protocol types of `simulator-api` as `simulator_client::simulator_api`, so you don't add that crate yourself. The full API reference is on [docs.rs](https://docs.rs/simulator-client/latest/simulator_client/), and runnable programs are in the starter code [repository](https://github.com/nitro-svm/examples).

{% hint style="info" %}
Each snippet below is a function, so it compiles on its own. Call it from a `main` like the one in [Run a session](#run-a-session).
{% endhint %}

## Install

```toml
[dependencies]
simulator-client = "0.29"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
rustls = { version = "0.23", default-features = false, features = ["ring"] }
```

The client needs Rust 1.97 or newer. If you add Solana crates, use these versions, so their types match the client's:

| Crate | Version |
| --- | --- |
| `solana-address` | 2.6 or newer, below 3 |
| `solana-pubkey` | 4.x |
| `solana-account`, `solana-client`, `solana-rpc-client`, `solana-rpc-client-api`, `solana-transaction`, `solana-transaction-status` | 4.3 or newer, below 5 |
| `solana-hash`, `solana-message` | 4.x |
| `solana-commitment-config`, `solana-system-interface` | 3.x |
| `solana-keypair`, `solana-signature` | 3.1 or newer, below 4 |
| `solana-loader-v3-interface` | 7.x |
| `solana-signer` | 3.0.1 or newer, below 4 |

Addresses are `simulator_client::Address`. A `Pubkey` from `solana-pubkey` 4 is the same type. Convert a `Pubkey` from an older Solana crate with `Address::new_from_array(pubkey.to_bytes())`.

### TLS

Install a rustls crypto provider at the top of `main`, before you connect:

```rust
let _ = rustls::crypto::ring::default_provider().install_default();
```

Without it, the first connection panics whenever anything else in your program enables rustls's `aws-lc-rs` provider (rustls's default features, `reqwest` 0.13, the AWS SDK).

### Logging

The client logs through `tracing`. Add `tracing-subscriber = { version = "0.3", features = ["env-filter"] }`, install a subscriber, and run with `RUST_LOG=simulator_client=info` to see session creation, reconnects and retries:

```rust
tracing_subscriber::fmt()
    .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
    .init();
```

## Managed Sessions

### Pick a Slot Range

List the slot ranges the server can replay, oldest first. A session starts at a range's `bundle_start_slot` and ends at or before its `max_bundle_end_slot`, which is `None` when the server reports no end. `filter_ranges` keeps the ranges between two slots or dates; `ranges.iter().max_by_key(|range| range.bundle_start_slot)` picks the newest.

```rust
use simulator_client::{AvailableRange, BacktestClient, RangeBound, filter_ranges};

async fn pick_range(
    host: &str,
    api_key: &str,
) -> Result<AvailableRange, Box<dyn std::error::Error>> {
    let client = BacktestClient::builder().url(host).api_key(api_key).build();
    let ranges = client.available_ranges().await?;

    let after: RangeBound = "2026-10-01".parse()?;
    let range = filter_ranges(&ranges, Some(&after), None)
        .into_iter()
        .next()
        .ok_or("no slot range starts after 2026-10-01")?;
    println!(
        "slots {} to {:?}",
        range.bundle_start_slot, range.max_bundle_end_slot
    );
    Ok(range)
}
```

`url` takes the server's host, such as `simulator.termina.technology`. The client sends the API key as the `X-API-Key` header. Read it from `SIMULATOR_API_KEY`, as `sim` does.

### Run a Session

Build the create request with `CreateSession`, turn it into a request with `.into_request()?`, and start the session. Then step it until it completes.

```rust
use simulator_client::{
    Address, BacktestClient, Continue, CreateSession, ManagedBacktestSession, ManagedEvent,
};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let _ = rustls::crypto::ring::default_provider().install_default();

    let host = "simulator.termina.technology";
    let api_key = std::env::var("SIMULATOR_API_KEY")?;

    // 1. Pick the newest slot range the server can replay. The list is oldest first.
    let client = BacktestClient::builder().url(host).api_key(&api_key).build();
    let ranges = client.available_ranges().await?;
    let range = ranges
        .iter()
        .max_by_key(|range| range.bundle_start_slot)
        .ok_or("the server lists no slot ranges")?;

    // 2. Start a session at the range's first slot. `slot_count` counts the slots after it.
    let create = CreateSession::builder()
        .start_slot(range.bundle_start_slot)
        .slot_count(10)
        .capacity_wait_timeout_secs(300)
        .build()
        .into_request()?;
    let mut session = ManagedBacktestSession::start(host.to_string(), api_key, create).await?;

    // 3. Read state just before the start slot executes (the end of the slot before it)
    //    through the session's RPC endpoint.
    session.wait_until_ready().await?;
    let usdc: Address = "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v".parse()?;
    let mint = session.rpc().get_account(&usdc).await?;
    println!("USDC mint: {} bytes, owned by {}", mint.data.len(), mint.owner);

    // 4. Step one block at a time until the session completes, then close it.
    let step = Continue::builder().advance_count(1).build().into_params();
    session.send_continue(step.clone()).await?;
    let summary = session
        .run_to_completion(step, |event| {
            if let ManagedEvent::Slot(slot) = event {
                println!("executed slot {slot}");
            }
        })
        .await?;
    println!("{summary:?}");
    Ok(())
}
```

Source: [`hello_session`](https://github.com/nitro-svm/examples/tree/main/src/bin/hello_session)

* Set exactly one of `end_slot` (inclusive) or `slot_count`. `into_request()` checks the builder before anything is sent.
* Set `capacity_wait_timeout_secs` for any unattended run. Unset, the create fails at once when the server is busy. `sim run` waits 900 seconds.
* Other fields: `actions` ([pre-scheduled actions](#pre-scheduled-actions)), `extra_compute_units` (raise each transaction's compute unit limit), `send_summary` (transaction statistics on completion), the `reroute_*` fields ([rerouting](#reroute-order-flow)), and more in the [full reference](https://docs.rs/simulator-client/latest/simulator_client/struct.CreateSession.html).
* To split a long range across concurrent sessions, set `parallel` and use `managed::ManagedParallelSession`.

How a session runs:

1. `start` returns once the server has created the session. `session_info()` holds the session ID and the session's RPC endpoint.
2. Call the `subscribe_*` methods you need, before the first continue.
3. `next_event` returns events. On `ManagedEvent::ReadyForContinue` the session is paused: read or change state, then call `send_continue` or `send_continue_to`, once per `ReadyForContinue`. After `send_continue_to`, the session sends `ManagedEvent::Paused` and then `ReadyForContinue`: answer only the `ReadyForContinue`.
4. `ManagedEvent::Completed` comes after the last slot and after every subscription's last notification.
5. `shutdown` closes the session on the server. `run_to_completion` does steps 3 to 5 for you, and stops on any `ManagedEvent::Error`, even one that leaves the session running.

Dropping a session closes it without waiting. If your program exits right after, the server keeps the session until its disconnect window runs out, so call `shutdown().await` when you stop early.

### Read Accounts

The session's RPC endpoint serves standard Solana JSON-RPC methods against the session's state, such as `getAccountInfo`, `getMultipleAccounts` and `getLatestBlockhash`. See [Session RPC and Subscriptions](api-reference/session-rpc.md) for the full list.

`session.rpc()` is a `solana_client` RPC client for that endpoint. From the first `ReadyForContinue` (after `wait_until_ready`) it serves the state before the start slot executes (the end of the slot before `start_slot`), and from every later one the state after each step. It serves until `Completed`, then answers HTTP 502.

* `get_account` fails with `AccountNotFound` for an account that doesn't exist. `get_multiple_accounts` returns `None` for it instead.

```rust
use simulator_client::{Address, ManagedBacktestSession, simulator_api::token};

/// The balance of an SPL token account, read while the session is paused.
async fn token_balance(
    session: &ManagedBacktestSession,
    token_account: Address,
) -> Result<Option<u64>, Box<dyn std::error::Error>> {
    let account = session.rpc().get_account(&token_account).await?;
    Ok(token::token_amount(&account.data))
}
```

### User Transactions

Add your own transactions to a continue. Before the next block executes, the server runs them in order and commits them, then writes the continue's account overrides.

* The server checks neither signatures nor the recent blockhash, so any key and any blockhash work, such as `Hash::default()`. A transaction that uses a durable nonce is the exception.
* The fee payer pays the fee, so it needs lamports. Fund it with an [account override](#account-overrides) on an earlier continue.

```rust
use simulator_client::{Continue, ManagedBacktestSession};
use solana_transaction::versioned::VersionedTransaction;

/// Run `tx`, then execute the next block. Call it on a `ReadyForContinue`.
async fn run_user_transaction(
    session: &mut ManagedBacktestSession,
    tx: &VersionedTransaction,
) -> Result<(), Box<dyn std::error::Error>> {
    let step = Continue::builder()
        .advance_count(1)
        .build()
        .push_transaction(tx)?
        .into_params();
    session.send_continue(step).await?;
    Ok(())
}
```

A failed user transaction does not produce an error event. Check its effect by reading accounts at the next `ReadyForContinue`.

#### Inject at a Slot

To run transactions just before slot `n` executes, pause at the end of slot `n - 1` with `send_continue_to`, then send them with the next continue:

```rust
use simulator_client::{Continue, ContinueToParams, ManagedBacktestSession, ManagedEvent};
use solana_transaction::versioned::VersionedTransaction;

/// Run `tx` just before slot `n`'s block. Call it on a `ReadyForContinue`.
async fn run_before_slot(
    session: &mut ManagedBacktestSession,
    n: u64,
    tx: &VersionedTransaction,
) -> Result<(), Box<dyn std::error::Error>> {
    let pause = ContinueToParams {
        slot: n - 1,
        batch_index: None,
    };
    session.send_continue_to(pause).await?;
    // The session sends `Paused`, then `ReadyForContinue`: answer only the second.
    loop {
        match session.next_event().await? {
            ManagedEvent::ReadyForContinue => break,
            ManagedEvent::Error(err) => return Err(err.into()),
            _ => {}
        }
    }
    let step = Continue::builder().build().push_transaction(tx)?.into_params();
    session.send_continue(step).await?;
    Ok(())
}
```

* `slot` must be after the last executed slot, or the session sends an `InvalidRequest` error and stays where it is.
* `slot` must have a block. The session doesn't stop at a skipped slot: it runs on to the end of the range. If slot `n - 1` may be skipped, schedule the transaction instead.
* `batch_index: Some(i)` pauses just before batch `i` of the slot, a batch being a group of the slot's transactions that execute together.

To skip the round trip, schedule the transaction when you create the session: a [pre-scheduled action](#pre-scheduled-actions) with `ActionAnchor::BeforeSlot` and `ActionKind::Send` runs and commits it at the start of slot `n`, before the slot's transactions.

```rust
use simulator_client::{
    encode_transaction,
    simulator_api::{AccountModifications, ActionAnchor, ActionKind, ScheduledAction},
};
use solana_keypair::Keypair;
use solana_message::Hash;
use solana_signer::Signer;
use solana_transaction::{Instruction, Transaction};

/// An action that runs `instruction`, paid by `payer`, at the start of `slot`.
fn send_at_slot(
    slot: u64,
    payer: &Keypair,
    instruction: Instruction,
) -> Result<ScheduledAction, Box<dyn std::error::Error>> {
    // Signatures and the blockhash aren't checked, so `Hash::default()` works.
    let tx = Transaction::new_signed_with_payer(
        &[instruction],
        Some(&payer.pubkey()),
        &[payer],
        Hash::default(),
    );
    Ok(ScheduledAction {
        anchor: ActionAnchor::BeforeSlot { slots: vec![slot] },
        kind: ActionKind::Send,
        transactions: vec![encode_transaction(&tx)?],
        account_overrides: AccountModifications::default(),
        feeds_reroute: false,
        return_accounts: Vec::new(),
        label: Some("inject".to_string()),
        resolve: None,
    })
}
```

Pass the action to `CreateSession::builder().actions(..)`. A skipped slot fires nothing.

### Pre-Scheduled Actions

An action runs your transactions at fixed slots, every N slots, or before or after each batch that matches a discovery filter, without a round trip per slot. Encode each transaction with `encode_transaction`, and read the results with `subscribe_actions`.

`ActionAnchor` says where it fires:

* `BeforeSlot { slots }`, `AfterSlot { slots }`: at the start or end of each listed slot. Transaction `i` fires at `slots[i]`.
* `BeforeEverySlot { every_n_slots }`, `AfterEverySlot { every_n_slots }`: at every slot that is a multiple of `every_n_slots`. All the transactions fire each time.
* `BeforeMatch { filter }`, `AfterMatch { filter }`: before or after each batch that matches a discovery filter.

A slot without a block fires nothing.

* `ActionKind::Simulate` runs against the session's state without committing. `ActionKind::Send` commits, so later slots see the writes. `account_overrides` work only with `Simulate`.
* `return_accounts` lists accounts whose state after the action you want back.
* As for user transactions, signatures and the recent blockhash aren't checked, and the fee payer needs lamports. Nothing rejects a duplicate: the same transaction can run again.

```rust
use std::num::NonZeroU64;

use simulator_client::{
    Address, Continue, CreateSession, ManagedBacktestSession, ManagedEvent, encode_transaction,
    simulator_api::{AccountModifications, ActionAnchor, ActionKind, ScheduledAction},
};
use solana_transaction::versioned::VersionedTransaction;

/// Simulate `tx` after every slot and print `wallet`'s lamports.
async fn probe_every_slot(
    host: &str,
    api_key: String,
    start_slot: u64,
    tx: &VersionedTransaction,
    wallet: Address,
) -> Result<(), Box<dyn std::error::Error>> {
    let probe = ScheduledAction {
        anchor: ActionAnchor::AfterEverySlot {
            every_n_slots: NonZeroU64::MIN,
        },
        kind: ActionKind::Simulate,
        transactions: vec![encode_transaction(tx)?],
        account_overrides: AccountModifications::default(),
        feeds_reroute: false,
        return_accounts: vec![wallet],
        label: Some("probe".to_string()),
        resolve: None,
    };
    let create = CreateSession::builder()
        .start_slot(start_slot)
        .slot_count(100)
        .capacity_wait_timeout_secs(300)
        .actions(vec![probe])
        .build()
        .into_request()?;
    let mut session = ManagedBacktestSession::start(host.to_string(), api_key, create).await?;
    session.subscribe_actions();

    let step = Continue::builder().advance_count(1).build().into_params();
    session
        .run_to_completion(step, |event| {
            let ManagedEvent::ActionResult(result) = event else {
                return;
            };
            if let Some(failure) = result.failure() {
                println!("slot {}: failed: {:?}", result.slot, failure.err);
            } else if let Some(Ok(account)) = result.returned_account(0) {
                println!("slot {}: wallet holds {} lamports", result.slot, account.lamports);
            }
        })
        .await?;
    Ok(())
}
```

Source: [`quote_decay`](https://github.com/nitro-svm/examples/tree/main/src/bin/quote_decay)

### Account Overrides

Overwrite accounts on a continue. Each override replaces the whole account. Build one with:

* `AccountData::system_account(lamports)` for a wallet holding SOL.
* `simulator_api::token::token_account(...)` for an SPL token account. To change the balance of an existing token account, pass it to `token::with_token_amount`.
* `AccountData::from(&account)` to copy an account read through `session.rpc()`.

```rust
use simulator_client::{
    Address, Continue, ManagedBacktestSession, ManagedSessionError,
    simulator_api::{
        AccountData, mints,
        token::{self, TokenProgram},
    },
};

/// Give `wallet` 10 SOL and 1,000 USDC, then execute the next block.
async fn fund_wallet(
    session: &mut ManagedBacktestSession,
    wallet: Address,
    usdc_account: Address,
) -> Result<(), ManagedSessionError> {
    let usdc = token::token_account(TokenProgram::Token, mints::USDC, wallet, 1_000_000_000);
    let step = Continue::builder()
        .advance_count(1)
        .build()
        .modify_account(wallet, AccountData::system_account(10_000_000_000))
        .modify_account(usdc_account, usdc)
        .into_params();
    session.send_continue(step).await
}
```

To give an action's transactions their own starting state, set `account_overrides` on the `ScheduledAction` instead. Overrides can't change a program: use a program override.

### Program Overrides

Test new program logic by replacing a program's binary with your own build. `modify_program_via_rpc` reads what it needs from the session's RPC endpoint and returns the account overrides to send on the next continue. The new binary runs from the next slot.

```rust
use simulator_client::{Continue, ManagedBacktestSession, modify_program_via_rpc};

/// Replace `program_id`'s binary with the one at `so_path`. Call it on a `ReadyForContinue`.
async fn override_program(
    session: &mut ManagedBacktestSession,
    program_id: &str,
    so_path: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    let elf = std::fs::read(so_path)?;
    let overrides = modify_program_via_rpc(session.rpc(), program_id, &elf).await?;
    let step = Continue::builder()
        .advance_count(1)
        .modify_accounts(overrides)
        .build()
        .into_params();
    session.send_continue(step).await?;
    Ok(())
}
```

Source: [`regression_test`](https://github.com/nitro-svm/examples/tree/main/src/bin/regression_test)

### Subscriptions

A managed session's subscriptions reconnect with it. Call them before the first continue, and their notifications arrive as events:

| Method | Event | Carries |
| --- | --- | --- |
| `subscribe_transactions(programs)` | `ManagedEvent::Transaction` | Each transaction that mentions one of the programs, as `getTransaction` returns it. Program logs are in `meta.log_messages`. |
| `subscribe_account_diffs(programs)` | `ManagedEvent::AccountDiff` | Each change to an account owned by one of the programs, before and after. |
| `subscribe_actions()` | `ManagedEvent::ActionResult` | Each [pre-scheduled action](#pre-scheduled-actions) result. |
| `subscribe_replacements()` | `ManagedEvent::Replacement` | Each [reroute](#reroute-order-flow) result, and the original swap replayed as the baseline. |
| `subscribe_diagnostics()` | `ManagedEvent::Diagnostic` | Each diagnostic probe sample from the create request. |

```rust
use simulator_client::{Address, Continue, ManagedBacktestSession, ManagedEvent};

/// Print every transaction's logs and every account change for `program`.
async fn stream_program(
    mut session: ManagedBacktestSession,
    program: Address,
) -> Result<(), Box<dyn std::error::Error>> {
    session.subscribe_transactions(vec![program]);
    session.subscribe_account_diffs(vec![program]);

    let step = Continue::builder().advance_count(1).build().into_params();
    session
        .run_to_completion(step, |event| match event {
            ManagedEvent::Transaction(tx) => {
                let tx = *tx;
                let logs: Option<Vec<String>> =
                    tx.transaction.meta.and_then(|meta| meta.log_messages.into());
                println!("slot {}: {:?}", tx.slot, logs);
            }
            ManagedEvent::AccountDiff(diff) => {
                println!("slot {}: {:?} changed", diff.context.slot, diff.account);
            }
            _ => {}
        })
        .await?;
    Ok(())
}
```

Source: [`counterfactual_flow`](https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_flow)

After a reconnect, a subscription sends the last slot it delivered again, so dedupe transactions by signature and account diffs by `(signature, account, slot)`.

The managed methods follow programs, not single accounts. To follow one account and keep reconnecting, subscribe to the program that owns it and keep the diffs whose `account` matches. For program logs, read `meta.log_messages` from `subscribe_transactions`.

The functions in `simulator_client::subscriptions` (`subscribe_account_diffs`, `subscribe_program_logs` and others) also follow single accounts and logs alone, on the session's RPC endpoint. They call your callback on a background task and don't reconnect: a dropped connection ends the stream with an error. Shut each one down before you close the session.

```rust
use simulator_client::{Continue, ManagedBacktestSession, ManagedEvent, subscribe_account_diffs};

/// Step the session to the end, printing every change to `account`.
async fn watch_account(
    mut session: ManagedBacktestSession,
    account: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    session.wait_until_ready().await?;
    let diffs = subscribe_account_diffs(
        &session.session_info().rpc_endpoint,
        account,
        |diff| async move {
            println!("slot {}: {:?}", diff.context.slot, diff.post);
        },
    )
    .await?;

    let step = Continue::builder().advance_count(1).build().into_params();
    session.send_continue(step.clone()).await?;
    loop {
        match session.next_event().await? {
            ManagedEvent::ReadyForContinue => {
                session.send_continue(step.clone()).await?;
            }
            ManagedEvent::Completed { .. } => break,
            ManagedEvent::Error(err) if err.is_terminal() => return Err(err.into()),
            _ => {}
        }
    }

    diffs.shutdown().await?;
    session.shutdown().await;
    Ok(())
}
```

Source: [`regression_test`](https://github.com/nitro-svm/examples/tree/main/src/bin/regression_test) (program logs)

### Reroute Order Flow

Reroute historical swaps to see how much flow a pool would've captured. There are two modes, and a session can run both:

* **Via an aggregator:** each original swap is re-quoted through the router at its [pricing point](../../flows/rerouting-historical-flow/README.md#interpreting-the-numbers). Add your pool with `reroute_extra_markets`, so the router considers it.
* **Directly to a venue:** each hop of an original swap is priced against your venue through a template. See [Rerouting Directly to a Venue](../../flows/rerouting-historical-flow/rerouting-directly-to-a-venue.md) for the template format.

`reroute_aggregators` picks whose swaps are rerouted. It defaults to Jupiter alone; `RerouteAggregators::all()` adds DFlow, OKX and Titan. Atomic arbitrage is not rerouted unless you set `reroute_circular_arbs`.

#### Via an Aggregator

To test a pool that doesn't exist onchain yet, add its accounts with `add_override` at the start slot. The easiest way is to copy an existing pool of the same program and change it.

{% hint style="warning" %}
If the program checks relationships between accounts (for example, vaults derived from the pool and mint), keep them consistent, or swaps through the new pool fail.
{% endhint %}

```rust
use std::collections::BTreeMap;

use simulator_client::{
    Address, Continue, CreateSession, ManagedBacktestSession, ManagedEvent,
    ReplacementNotification,
    reroute_report::{self, Target},
    simulator_api::{AccountData, AccountModifications, RerouteAggregators},
};

/// Reroute every aggregator's swaps with `pool` available, then report the flow it won.
/// `pool_accounts` holds the pool and its vaults.
async fn reroute_via_aggregator(
    host: &str,
    api_key: String,
    start_slot: u64,
    end_slot: u64,
    pool: Address,
    pool_program: Address,
    pool_accounts: BTreeMap<Address, AccountData>,
) -> Result<(), Box<dyn std::error::Error>> {
    let create = CreateSession::builder()
        .start_slot(start_slot)
        .end_slot(end_slot)
        .capacity_wait_timeout_secs(900)
        .send_summary(true)
        .reroute_order_flow(true)
        .reroute_aggregators(RerouteAggregators::all())
        .reroute_extra_markets([pool].into())
        .build()
        .add_override(start_slot, AccountModifications(pool_accounts))
        .into_request()?;
    let mut session = ManagedBacktestSession::start(host.to_string(), api_key, create).await?;
    session.subscribe_replacements();

    let mut notifications: Vec<ReplacementNotification> = Vec::new();
    let step = Continue::builder().advance_count(100).build().into_params();
    let summary = session
        .run_to_completion(step, |event| {
            if let ManagedEvent::Replacement(notification) = event {
                notifications.push(*notification);
            }
        })
        .await?;

    // Identify the venue by its router label, its program ID, or both.
    let target = Target::new(None, Some(pool_program));
    let report = reroute_report::from_notifications(target, &notifications)?;
    println!("{}", report.render(None));

    if let Some(stats) = summary.and_then(|summary| summary.reroute_stats) {
        println!(
            "swaps detected {}, rerouted {}, simulated {}, succeeded {}",
            stats.swaps_detected, stats.swaps_rerouted, stats.swaps_simulated, stats.swaps_succeeded
        );
    }
    Ok(())
}
```

Source: [`counterfactual_flow`](https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_flow)

Each `ReplacementNotification` is one of:

* `Requote`: a reroute via an aggregator, with the route the router chose hop by hop, the amounts in and out, and the original route and fill onchain.
* `DirectFill`: a reroute directly to a venue, priced through one template.
* `Original`: the original swap, replayed for comparison.

#### Directly to a Venue

Turn off rerouting via an aggregator with `reroute_requote(false)` and pass your templates. Reading `template.json` here needs `serde_json`.

```rust
use simulator_client::{
    CreateSession, DirectFillTemplate, ReplacementNotification, reroute_report,
    simulator_api::{CreateBacktestSessionRequest, RerouteAggregators},
};

/// A create request that prices every aggregator's swaps against the templates in `template.json`.
fn reroute_to_venue(
    start_slot: u64,
    end_slot: u64,
) -> Result<CreateBacktestSessionRequest, Box<dyn std::error::Error>> {
    let templates: Vec<DirectFillTemplate> =
        serde_json::from_str(&std::fs::read_to_string("template.json")?)?;
    Ok(CreateSession::builder()
        .start_slot(start_slot)
        .end_slot(end_slot)
        .capacity_wait_timeout_secs(900)
        .reroute_order_flow(true)
        .reroute_requote(false)
        .reroute_direct_fill(templates)
        .reroute_aggregators(RerouteAggregators::all())
        .build()
        .into_request()?)
}

/// The per-template report from the notifications `subscribe_replacements` delivered.
fn print_venue_report(notifications: &[ReplacementNotification]) {
    let probes = reroute_report::probes_from_notifications(notifications);
    println!("{}", probes.render("my-venue"));
}
```

Source: [`reroute_venue_templates`](https://github.com/nitro-svm/examples/tree/main/src/bin/reroute_venue_templates)

### Connection Drops

The session runs on the server and outlives its connections, unless `disconnect_timeout_secs` is 0. A managed session retries a dropped connection up to 20 times within 5 minutes, reattaches, and the server replays the responses you missed. You don't need to do anything.

* The server keeps an unattached session for `disconnect_timeout_secs`. Unless you set it, a managed single session uses 360 seconds and a parallel run the server's 900 (at most 900).
* Close code 1012 means the server paused the session for a reattach, for example during a redeploy. The managed session reattaches on its own.

### Errors

The managed session has already retried what it could, so retrying the same session never helps. Each `ManagedSessionError` says what to do, and `is_retryable()` answers the last column in code:

| Error | Cause | What to do | Retry with a new session? |
| --- | --- | --- | --- |
| `ApiKeyRejected` | The server refused the API key (HTTP 401 or 403). Never retried. | Fix the key. `sim usage` confirms a key works. | No |
| `Create` | The server refused the create request (`CreateError::Rejected`), couldn't be reached (`CreateError::Connection`), or sent a reply this client doesn't understand (`CreateError::Protocol`). | Retry later, fix the request, or upgrade `simulator-client`. | When `CreateError::is_retryable()` |
| `Remote` | The server reported an error, terminal or not, during `wait_until_ready` or `run_to_completion`. | Fix what it names. | When `BacktestError::is_retryable()` |
| `ControlFailed`, `SubscriptionFailed` | A connection ran out of retries. | Start a new session. | Yes |
| `ControlClosed`, `ContinueSend` | The session is over. | Stop. | No |
| `Cancelled` | Your cancellation token fired, or the parallel run was shut down. | Stop. | No |
| `CompletedBeforeReady` | The session finished before its first pause. | Check the range and the create request. | No |

A busy server fails the create with `CreateError::Rejected` holding `BacktestError::Internal` (`unable to start session: insufficient capacity`), which is retryable. There is no per-key limit on sessions.

`ManagedEvent::Error` from `next_event` ends the session only when `err.is_terminal()` is true. Otherwise the session keeps running. A transaction that fails during the replay is logged, not reported as an error.

Common messages:

| Message | Fix |
| --- | --- |
| `API key rejected: HTTP 401` (or 403), or `401 Unauthorized` from `available_ranges` | Check `SIMULATOR_API_KEY`. `sim usage` confirms a key works. |
| `session create failed: ... block bundle not found`, `invalid request` or `no produced blocks in requested range` | Pick a range from `available_ranges` and start at its `bundle_start_slot`. |
| `session create failed: ... insufficient capacity` | Set `capacity_wait_timeout_secs`, or retry later. |
| Panic: `Could not automatically determine the process-level CryptoProvider` | Install a provider: see [TLS](#tls). |
| Expected `Address`, found `Pubkey` | Use the versions in [Install](#install). `cargo tree -i solana-address@2 -e normal` shows which crates use the client's version. Other major versions of `solana-address`, pulled in by `spl-*` crates, are harmless. |

## Raw Sessions

The raw layer is the protocol itself: one WebSocket per session and no reconnect. Use it to pause on discovery filters, or to drive the protocol yourself.

### Create and Step

`BacktestClient::create_session` takes the `CreateSession` builder itself. Always set `disconnect_timeout_secs`: unset, the server waits only 30 seconds for you to reattach.

```rust
use simulator_client::{Address, BacktestClient, Continue, CreateSession};

async fn raw_session(
    host: &str,
    api_key: &str,
    start_slot: u64,
) -> Result<(), Box<dyn std::error::Error>> {
    let client = BacktestClient::builder().url(host).api_key(api_key).build();
    let mut session = client
        .create_session(
            CreateSession::builder()
                .start_slot(start_slot)
                .slot_count(10)
                .capacity_wait_timeout_secs(300)
                .disconnect_timeout_secs(300)
                .build(),
        )
        .await?;

    if session.ensure_ready(None).await?.is_completed() {
        return Ok(());
    }
    let usdc: Address = "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v".parse()?;
    let mint = session.rpc().get_account(&usdc).await?;
    println!("USDC mint: {} bytes", mint.data.len());

    loop {
        let step = Continue::builder().advance_count(1).build();
        let result = session.continue_until_ready(step, None, |_| {}).await?;
        if result.completed {
            break;
        }
    }
    session.close(None).await?;
    Ok(())
}
```

`BacktestSession` also has `modify_program` for [program overrides](#program-overrides), and `subscribe_program_logs` and `subscribe_account_diffs` for the session's RPC endpoint.

### Discovery Filters

Register filters at creation, such as "a transaction invokes this program". `advance_to_discovery` pauses the session just before each matching batch, so you can read the state that batch will execute against.

```rust
use std::time::Duration;

use simulator_client::{
    Address, BacktestClient, CreateSession, DiscoveryFilter, DiscoveryStepResult,
    decode_transaction,
};

/// Pause before every batch that invokes `program`.
async fn pause_before_program(
    client: &BacktestClient,
    start_slot: u64,
    end_slot: u64,
    program: Address,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut session = client
        .create_session(
            CreateSession::builder()
                .start_slot(start_slot)
                .end_slot(end_slot)
                .capacity_wait_timeout_secs(300)
                .disconnect_timeout_secs(300)
                .discoveries(vec![DiscoveryFilter::ProgramExecuted(program)])
                .build(),
        )
        .await?;
    session.ensure_ready(None).await?;

    let timeout = Some(Duration::from_secs(120));
    while let DiscoveryStepResult::Paused(pause) =
        session.advance_to_discovery(None, timeout).await?
    {
        // Paused before the batch: `session.rpc()` reads the state it will execute against.
        for encoded in &pause.discovery.transactions {
            let tx = decode_transaction(encoded)?;
            println!(
                "slot {} batch {}: {:?}",
                pause.paused.slot,
                pause.discovery.batch_index,
                tx.signatures.first()
            );
        }
    }
    session.close(None).await?;
    Ok(())
}
```

Source: [`quote_compare`](https://github.com/nitro-svm/examples/tree/main/src/bin/quote_compare)

A managed session delivers the same pauses as `ManagedEvent::DiscoveryBatch`: answer with `send_continue_to`, and read state on `ManagedEvent::Paused`.

### Reattach

On the raw layer, treat any close or transport error as "reattach":

1. Call `attach_session` with the session ID and the last sequence number you received. The server replays every response after it.
2. Open your subscriptions again.
3. Call `resume_attached_session`. A session the server paused for a reattach stays paused until you do.

Reattach within the session's `disconnect_timeout_secs`, or the server ends the session.

```rust
use simulator_client::{BacktestClient, BacktestSession};

async fn reattach(
    client: &BacktestClient,
    lost: &BacktestSession,
) -> Result<BacktestSession, Box<dyn std::error::Error>> {
    let session_id = lost.session_id().ok_or("the session was never created")?;
    let mut session = client
        .attach_session(session_id, lost.last_sequence())
        .await?;
    // Open your subscriptions again here.
    session.resume_attached_session().await?;
    Ok(session)
}
```

### Raw Errors

Each `BacktestClientError` variant says whether to retry, reattach or stop. The main ones:

| Error | What to do |
| --- | --- |
| `InvalidParams` | Fix the builder setting it names. Nothing was sent. |
| `Connect` | With HTTP 401 or 403, fix the API key. Otherwise retry with backoff. |
| `Remote` | The server refused. If `is_terminal()` is true, stop. `SessionNotFound` on attach means the session ended: start a new one, and raise `disconnect_timeout_secs`. |
| `Closed`, `WebSocket` | The connection is gone. Reattach. |
| `Timeout` | Retry, or raise the timeout. |
| `Http` | Retry a network failure with backoff. A body that doesn't decode means the client is older than the server: upgrade it. |
| `HttpStatus` | A non-2xx answer from an HTTP call, such as `available_ranges`. 401 or 403: fix the key. 429 or 5xx: retry later. Other 4xx: read `body`. |

`BacktestClientError::is_retryable()` says whether the same call may succeed after a backoff.

## Bundles and Datasets

`BacktestClient` also calls the server's HTTP endpoints. Each returns the response body described in [HTTP Endpoints](api-reference/http-endpoints.md):

| Call | Endpoint |
| --- | --- |
| `build_bundle`, `get_bundle_status`, `list_bundle_builds`, `delete_bundle` | [Bundles](api-reference/http-endpoints.md#bundles) |
| `generate_depth`, `get_depth_status`, `get_depth_coverage` | [Depth Datasets](api-reference/http-endpoints.md#depth-datasets) |
| `dataset_credentials` | [`POST /datasets/credentials`](api-reference/http-endpoints.md#post-datasets-credentials) |
| `usage`, `usage_sessions` | [Usage](api-reference/http-endpoints.md#usage) |

```rust
use simulator_client::{BacktestClient, BundleBuildStatus, DepthRequest, DepthSource};

/// Request SOL-USDC onchain depth for a slot range, and print where it lands.
async fn request_depth(
    client: &BacktestClient,
    start_slot: u64,
    end_slot: u64,
) -> Result<(), Box<dyn std::error::Error>> {
    let request = DepthRequest {
        start_slot,
        end_slot,
        source: DepthSource::Onchain,
        markets: vec!["SOL-USDC".to_string()],
        ..Default::default()
    };
    let job = client.generate_depth(&request).await?;
    // Poll `get_depth_status` until the job is no longer in progress.
    let status = client.get_depth_status(&job.request_id).await?;
    if status.status == BundleBuildStatus::Completed {
        let credentials = client.dataset_credentials().await?;
        println!("published under {:?}", credentials.user_s3_uri);
    }
    Ok(())
}
```

Source: [`datasets`](https://github.com/nitro-svm/examples/tree/main/src/bin/datasets)
