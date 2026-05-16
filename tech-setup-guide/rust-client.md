# Rust Client

The `simulator-client` and `simulator-api` crates provide a native Rust interface to the Termina simulator, for teams who want to integrate backtesting directly into their Rust code rather than using the `sim` CLI.

* **`simulator-client`** is the high-level async client. It wraps the WebSocket protocol with ergonomic builders for common workflows: creating sessions, advancing slots, injecting transactions, and reading account state.
* **`simulator-api`** defines the raw protocol types (request/response structs, error variants, session parameters). Use it if you need direct access to the wire format or want to implement your own client.

### Installation

```toml
[dependencies]
simulator-client = "0.7"
simulator-api = "0.7" # If you only need the protocol types (e.g. to build a custom client):
```

### Examples

#### Available Slots

Before creating a session, confirm the slot range you want to replay is available:

```rust
let ranges = client.available_ranges().await?;
for r in &ranges {
    println!(
        "slots {} – {}",
        r.bundle_start_slot,
        r.max_bundle_end_slot.unwrap_or(0)
    );
}
```

#### Program Overrides

Load a compiled ELF binary and patch it into the session.

```rust
let elf = std::fs::read("your_program.so")?;

// Derives the correct ProgramData account shape via the session's RPC endpoint
let modifications = session
    .modify_program("YourProgramId111111111111111111111111111111", &elf)
    .await?;

// Apply the modifications on the next Continue
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

#### Reads + Writes

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

#### Subscriptions

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

#### Session Summaries

Enable `send_summary` at session creation to receive transaction-level statistics on completion. The `Completed` response will include a `SessionSummary` with counts of correct simulations, mismatches, balance diffs, and execution errors.

```rust
let mut session = client
    .create_session(
        CreateSession::builder()
            .start_slot(300_000_000)
            .slot_count(100)
            .send_summary(true)
            .build(),
    )
    .await?;
```
