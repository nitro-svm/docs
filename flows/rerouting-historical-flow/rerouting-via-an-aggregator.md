# Rerouting via an Aggregator

{% stepper %}
{% step %}
### Reroute Flow

To reroute order flow through Jupiter Metis and evaluate the performance of a program's new quoting logic, enable the reroute flag on the [`sim run` command](../replaying-the-market.md#run-a-baseline).

```bash
sim run \
  --start-slot 400000000 \
  --end-slot 400010000 \
  --program-id <PROGRAM_ID> \
  --program-so <NEW_QUOTING_LOGIC_BINARY> \
  --reroute-order-flow router
```
{% endstep %}

{% step %}
### Select Aggregators

The simulation can match against swaps from the major aggregators: Jupiter, DFlow, OKX, and Titan.&#x20;

By default, only Jupiter transactions are rerouted through Metis, since that gives the most faithful replay. But swaps from other routers can be rerouted through Metis as well.

```bash
sim run ...
    --reroute-aggregators jupiter,okx
```

For example, if a market maker isn't integrated with DFlow or Titan and can't win flow from them on mainnet, they can choose to enable only Jupiter and OKX in the simulation.
{% endstep %}

{% step %}
### Toggle Arbitrage

A meaningful portion of aggregator volume is toxic flow, especially atomic arbitrage: a circular loop like `SOL` → `USDC` → `SOL` within a single transaction.

Most arbs are built by searchers outside the aggregator and only wrapped in the aggregator's instructions. That means rerouting them through Metis often won't reflect how they were actually constructed.

By default, atomic arbs are not rerouted and keep their original route, but they can be toggled on. When enabled, the simulation looks for a new route that still preserves the original swap intent: more concretely, a `SOL` → `USDC` → `SOL` arb may have originally gone through venue A but can be rerouted to go through venue B.

```bash
sim run ...
    --reroute-circular-arbs
```
{% endstep %}

{% step %}
### Customize Market Conditions&#x20;

The `sim` CLI client currently only supports changes to the program binary.&#x20;

To test other parameters, like an oracle update's fair value or its position in a block, or the amount of capital that's deployed in the pool vaults, use the [Rust client](../../overview/installation/rust-reference.md). The [public examples](https://github.com/nitro-svm/examples) are a good place to start.
{% endstep %}
{% endstepper %}
