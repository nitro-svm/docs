# Rust Reference

The `simulator-client` and `simulator-api` crates provide a native Rust interface to the Termina simulator, for teams who want to integrate backtesting directly into their Rust code rather than using the `sim` CLI.

* **`simulator-client`** is the high-level async client. It wraps the WebSocket protocol with ergonomic builders for common workflows: creating sessions, advancing slots, injecting transactions, and reading account state.
* **`simulator-api`** defines the raw protocol types (request/response structs, error variants, session parameters). Use it if you need direct access to the wire format or want to implement your own client.

For a complete set of end-to-end examples, see the starter code [repository](https://github.com/nitro-svm/examples).

### Installation

```toml
[dependencies]
simulator-client = "0.23"
simulator-api = "0.23" # Only if you require protocol types (e.g. to build a custom client)
```

### Examples

#### Available Slots

Before creating a session, confirm the slot range you want to backtest is available.

```rust
use simulator_client::BacktestClient;

let client = BacktestClient::builder()
    .url("wss://simulator.termina.technology/backtest")
    .api_key(api_key)
    .build();

let ranges = client.available_ranges().await?;
for r in &ranges {
    println!(
        "slots {} – {}",
        r.bundle_start_slot,
        r.max_bundle_end_slot.unwrap_or(0)
    );
}
```

#### Session Initialization

Set the slot range and any optional fields in the session creation parameter, then create the session:

* `signer_filter`: skip historical transactions signed by these addresses
* `actions`: run a pre-scheduled action at every slot or triggered by specific transactions
* `extra_compute_units`: bump the compute unit cap to prevent transaction failures

...and others in the [full reference](https://docs.rs/simulator-api/0.23.0/simulator_api/struct.CreateSessionParams.html).

```rust
use simulator_client::{Continue, CreateSession, ManagedBacktestSession, ManagedEvent};

let request = CreateSession::builder()
    .start_slot(300_000_000)
    .slot_count(100)
    .build();
```

```rust
// Use the managed client instead of the raw `BacktestClient`,
// since it automatically retries if connections are dropped
let mut session = ManagedBacktestSession::start(
    "wss://simulator.termina.technology/backtest".to_string(),
    api_key,
    request,
).await?;

loop {
    // The server emits an initial `slotNotification` for `startSlot`, 
    // followed by `readyForContinue` when the session is ready
    match session.next_event().await? {
        ManagedEvent::ReadyForContinue => {
            session.send_continue(Continue::builder().build().into_params()).await?;
        }
        ManagedEvent::Completed { .. } => break,
        _ => {}
    }
}
session.shutdown().await;
```

#### Reads + Writes

Each session supports standard Solana JSON-RPC methods and subscriptions at `/backtest/<session_id>`, including:

* `getAccountInfo`, `getBalance`, `getMultipleAccounts`
* `accountSubscribe`, `programSubscribe`, `signatureSubscribe`&#x20;

> See [API Reference](api-reference.md#api-table) for the full list of supported Solana subscription methods.

Use the session's RPC client to send transactions and read accounts.&#x20;

```rust
use std::str::FromStr;
use solana_sdk::pubkey::Pubkey;

// Build your transaction using any Solana SDK tooling
let tx: solana_sdk::transaction::VersionedTransaction = /* ... */;

session
    .continue_until_ready(
        Continue::builder()
            .advance_count(1)
            .build()
            .push_transaction(&tx)?,
        Some(Duration::from_secs(30)),
        |_| {},
    )
    .await?;

let pubkey = Pubkey::from_str("SomePubkey11111111111111111111111111111111")?;
let account = session.rpc().get_account(&pubkey).await?;
println!("lamports: {}", account.lamports);
```

Or establish a websocket connection to the standard subscription methods.

```rust
use solana_commitment_config::CommitmentConfig;

let _handle = session
    .subscribe_program_logs(
        "YourProgramId111111111111111111111111111111",
        CommitmentConfig::confirmed(),
        |notification| async move {
            println!("logs: {:?}", notification.value.logs);
        },
    )
    .await?;
    
let _handle = session
    .subscribe_account_diffs(
        "SomePubkey11111111111111111111111111111111",
        |diff| async move {
            println!("account changed: {:?}", diff);
        },
    )
    .await?;

// Drop the handle to unsubscribe
```

#### Program Overrides

Test new program logic by compiling a new binary and overriding the existing one.

```rust
let elf = std::fs::read("your_program.so")?;

// Derives the correct ProgramData account shape via the session's RPC endpoint
let modifications = session
    .modify_program("YourProgramId111111111111111111111111111111", &elf)
    .await?;

// Apply the modifications on the next `Continue`
session
    .continue_until_ready(
        Continue::builder()
            .advance_count(1)
            .modify_accounts(modifications)
            .build(),
        None,
        |_| {},
    )
    .await?;
```

#### Rerouted Order Flow

Reroute historical swaps through Jupiter Metis to see how changes to parameters affect the flow a pool would've captured.

```rust
use simulator_client::{AccountModifications, CreateSession};

// If needed, inject a pool that doesn't yet exist onchain to test quoting for a new market
// (The easiest approach is to copy and tweak an existing pool owned by the same program)
let accounts = AccountModifications(BTreeMap::from([
    (pool, AccountData {
        data: EncodedBinary::new(pool_data_b64, BinaryEncoding::Base64),
        owner: pool_program,
        lamports: pool_lamports,
        executable: false,
        space: pool_data_len,
    }),
    (vault_a, /* SPL token account holding inventory */),
    (vault_b, /* SPL token account holding inventory */),
]));
```

{% hint style="warning" %}
When injecting a new pool, if the program checks relationships between accounts (e.g. vaults derived from the pool and mint), keep those consistent, or swaps through the new pool will fail.
{% endhint %}

Update the session creation parameters to enable rerouting.

```rust
use simulator_api::{RerouteAggregators, SwapAggregator};

let all_aggregators = RerouteAggregators::new([
    SwapAggregator::Jupiter, 
    SwapAggregator::Okx,
    SwapAggregator::Dflow,
    SwapAggregator::Titan,
]);

let request = CreateSession::builder()
    .start_slot(START_SLOT)
    .end_slot(END_SLOT)
    .reroute_order_flow(true)               // Reroute historical swaps through Jupiter Metis
    .reroute_aggregators(all_aggregators)   // Filter which aggregators' swaps to reroute
    .reroute_extra_markets([pool].into())   // Make the router aware of your pool
    .build()
    .add_override(START_SLOT, accounts)     // Set your pool existence from the first slot
    .into_request()?;
```

#### Summary + Measurements

To get a summary of the reroutes, ubscribe to `replacementSubscribe` notifications. Each rerouted swap includes:

* the route the router chose, hop by hop, with the address of each pool used
* amounts in and out for each hop
* the original route and fill onchain, for comparison

```rust
use simulator_client::reroute_report::{self, Target};

// Identify the venue by its router label, its program ID, or both
let target = Target::new(None, Some(pool_program));
let report = reroute_report::from_notifications(target, &notifications)?;
println!("{}", report.render(None));
```

The session summary also reports how many swaps were detected, rerouted, simulated and succeeded.
