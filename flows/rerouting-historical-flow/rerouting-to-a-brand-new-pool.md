# Rerouting to a Brand New Pool

Test how aggregator would have routed historical order flow if your pool had existed. You inject a pool that doesn't exist on-chain, make it visible to the router, and replay real swaps to see how much flow it would have captured.

{% hint style="info" %}
This workflow uses the Rust client (`simulator-client`). It isn't available in the `sim` CLI, because the pool accounts are built in code.
{% endhint %}

### Steps

#### 1. Set up a Rust project

```bash
cargo new my-pool-sim && cd my-pool-sim
cargo add simulator-client tokio --features tokio/full
```

#### 2. Pick a slot range

```bash
sim ranges --api-key <KEY> --after <YYYY-MM-DD>
```

Choose a range where your pair trades actively.

#### 3. Build your pool accounts

The easiest approach is to copy an existing pool of the same program and edit it:

1. Fetch an existing pool's account data with `getAccountInfo`. Use it as a template.
2. Edit the fields for your setup: mints, vault addresses, and price or inventory parameters.
3. Create the vault token accounts, with balances set to the inventory you want to test.
4. Keep the owner as your pool's program, and keep every account rent-exempt.

{% hint style="warning" %}
If the program checks relationships between accounts (for example, vaults derived from the pool and mint), keep those consistent, or swaps through your pool will fail.
{% endhint %}

#### 4. Package the accounts as an override

```rust
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

#### 5. Build the session request

```rust
let request = CreateSession::builder()
    .start_slot(START_SLOT)
    .end_slot(END_SLOT)
    .reroute_order_flow(true)               // re-quote historical swaps through Jupiter's router
    .reroute_extra_markets([pool].into())   // make the router aware of your pool
    .reroute_aggregators(/* jupiter, okx, titan, dflow */)
    .send_summary(true)
    .build()
    .add_override(START_SLOT, accounts)     // your pool exists from the first slot
    .into_request()?;
```

| Setting                       | Purpose                                                                                                  |
| ----------------------------- | -------------------------------------------------------------------------------------------------------- |
| `reroute_order_flow(true)`    | Re-quotes each historical swap through Jupiter's router                                                  |
| `reroute_extra_markets`       | Adds your pool to the markets the router considers                                                       |
| `reroute_aggregators`         | Selects which historical swaps to re-route; include all four to cover all flow                           |
| `add_override(START_SLOT, …)` | Injects your pool's accounts at the first slot. This is required for a pool that doesn't exist on-chain. |

#### 6. Run the session

```rust
let mut session = ManagedBacktestSession::start(
    "wss://simulator.termina.technology/backtest".to_string(),
    api_key,
    request,
).await?;

loop {
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

#### 7. Collect the results

Subscribe to `replacementSubscribe` notifications. Each re-quoted swap includes:

* the route the router chose, hop by hop, with the address of each pool used
* amounts in and out for each hop
* the original route and fill on-chain, for comparison

#### 8. Measure your pool's flow

* **Directly:** count the re-quoted swaps whose route includes your pool address, and sum the volume through it.
* **With the built-in report:**

```rust
let target = Target::new(None, Some(pool_program));
let report = reroute_report::from_notifications(target, &notifications)?;
println!("{}", report.render(None));
```

The session summary also reports how many swaps were detected, rerouted, simulated and succeeded.

### What this measures

This estimates the flow Jupiter's router would have sent to your pool, including split and multi-hop routes, compared with how the same swaps were actually routed.
