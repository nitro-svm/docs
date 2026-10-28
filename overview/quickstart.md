# Quickstart

Run your first session over a few historical slots. Pick the path that fits you:

* **CLI**: `sim` from a shell. No code.
* **Rust**: the `simulator-client` crate.
* **Raw**: JSON over a WebSocket, from any language.

All three need an API key. [Request](https://t.me/rustemzzzz) one if you don't have it yet.

{% tabs %}
{% tab title="CLI" %}
#### 1. Install and Check Your Key

Install `sim` as shown in [Installation](installation/README.md#cli), then set your key and check it:

```bash
export SIMULATOR_API_KEY=<API_KEY>
sim usage
```

`sim usage` prints your key's usage. A rejected key prints an `HTTP 401` error.

#### 2. Pick a Slot Range

List the slot ranges the server can replay:

```bash
sim ranges --kind transactions
```

Pick a row. Use its Start Slot as `<START_SLOT>`, and a slot about 100 later, at or before its End Slot, as `<END_SLOT>`.

#### 3. Run

Replay the range and record the transactions of one program. This example records Jupiter's; use your own program's ID once it works.

```bash
sim run \
  --start-slot <START_SLOT> \
  --end-slot <END_SLOT> \
  --program-id JUP6LkbZbjS1jKKwapdHNy74zcZ3tLUZoi5QNyVTaV4 \
  --output-file baseline.ndjson
```

* Without `--program-id`, the run records no transactions.
* If the program ran no transaction in the range, the run ends with `no matching transactions found in the requested slot range` and exit status 1. Pick another range.
* Press Ctrl+C once to stop early and keep what was recorded. `sim` then exits with status 130, and the file has no `summary` line.

#### 4. Read the Results

```bash
sim summarize baseline.ndjson
```

The file is newline-delimited JSON, one object per line. Each `tx` line is one transaction:

* `success` and `error` are its outcome in the replay. Without overrides the replay matches onchain, so a failed transaction usually failed onchain too.
* `sol_changes` and `token_changes` list only the accounts whose balance changed. SOL changes include the fee.

For example, list the failed transactions and their errors with [`jq`](https://jqlang.org/download/):

```bash
jq -c 'select(.type == "tx" and (.data.success | not)) | .data | {signature, error}' baseline.ndjson
```

Next:

* [Replaying the Market](../flows/replaying-the-market.md): run your own program binary against the same slots and compare it with this baseline.
* [Output Format](../cli/output-format.md): every field in the output file.
* [CLI](../cli/README.md): all commands, environment variables and exit codes.
{% endtab %}

{% tab title="Rust" %}
#### 1. Add the Dependencies

```toml
[dependencies]
simulator-client = "0.29"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
rustls = { version = "0.23", default-features = false, features = ["ring"] }
```

The client needs Rust 1.97 or newer.

#### 2. Write `main.rs`

The program picks a slot range, starts a session, reads one account and steps the session to the end.

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

* The first line installs a TLS crypto provider. Without it, the first connection can panic.
* `capacity_wait_timeout_secs` waits up to 300 seconds when the server is busy, instead of failing at once.
* The session reconnects and reattaches on its own if the connection drops.
* `run_to_completion` closes the session when it ends. If a step fails first, `?` drops the session, which closes it without waiting; if the program exits right away, the server keeps the session for up to 360 seconds.

#### 3. Run It

```bash
export SIMULATOR_API_KEY=<API_KEY>
cargo run
```

It prints the USDC mint's size and owner, one line per executed slot, then the session summary.

Source: [`hello_session`](https://github.com/nitro-svm/examples/tree/main/src/bin/hello_session)

Next:

* [Rust Reference](installation/rust-reference.md): user transactions, account and program overrides, subscriptions and rerouting.
* [docs.rs](https://docs.rs/simulator-client/latest/simulator_client/): every type and method.
* [Starter code](https://github.com/nitro-svm/examples): complete programs for each use case.
{% endtab %}

{% tab title="Raw" %}
#### 1. Pick a Slot Range

```bash
curl -s -H "X-API-Key: $SIMULATOR_API_KEY" https://simulator.termina.technology/available-ranges
```

Take a range's `bundle_start_slot` as `startSlot`, and an `endSlot` at or before its `max_bundle_end_slot`.

#### 2. Connect

Open a WebSocket to the session control channel at `wss://simulator.termina.technology/backtest`, with the header `X-API-Key: <API_KEY>`. For example, with [websocat](https://github.com/vi/websocat):

```bash
websocat -H "X-API-Key: $SIMULATOR_API_KEY" wss://simulator.termina.technology/backtest
```

Keep this connection open for the whole session.

#### 3. Create the Session

In the transcripts below, `>` is a message you send and `<` one the server sends.

```
> {"method":"createBacktestSession","params":{"startSlot":380000000,"endSlot":380000001,"disconnectTimeoutSecs":300}}
< {"method":"status","params":{"status":"startingRuntime"}}
< {"method":"sessionCreated","params":{"session_id":"backtest_019a2b3c…","rpc_endpoint":"/backtest/backtest_019a2b3c…","task_id":"…","session_name":"…"}}
< {"seqId":0,"method":"readyForContinue"}
< {"seqId":1,"method":"slotNotification","params":379999999}
```

* Request params are camelCase; `sessionCreated` fields are snake_case.
* Keep `session_id` and the last `seqId`: you need them to [reattach](installation/api-reference/raw-websocket-walkthrough.md#reattach).
* The session's RPC endpoint is the server's host plus `rpc_endpoint`. It serves standard Solana JSON-RPC, such as `getAccountInfo`, against the session's state.

#### 4. Execute Blocks

Send `continue`, then wait for `readyForContinue` before the next one.

```
> {"method":"continue","params":{"advanceCount":1}}
< {"seqId":2,"method":"status","params":{"status":"appliedAccountModifications"}}
< …
< {"seqId":6,"method":"slotNotification","params":380000000}
< {"seqId":7,"method":"success"}
< {"seqId":8,"method":"readyForContinue"}
```

The last block ends with `completed` instead of `readyForContinue`.

#### 5. Close

```
> {"method":"closeBacktestSession"}
< {"method":"success"}
```

Next:

* [Raw WebSocket Walkthrough](installation/api-reference/raw-websocket-walkthrough.md): the full transcript, RPC reads and reattaching.
* [API Reference](installation/api-reference/README.md): every request, response, status and error.
{% endtab %}
{% endtabs %}
