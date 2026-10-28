# Rerouting via an Aggregator

{% stepper %}
{% step %}
### Reroute Flow

To reroute order flow through an aggregator router and evaluate the performance of a program's new quoting logic, enable the reroute flag on the [`sim run` command](../replaying-the-market.md#run-a-baseline). Currently, the available router is Jupiter Metis.

```bash
sim run \
  --start-slot <START_SLOT> \
  --end-slot <END_SLOT> \
  --program-id <PROGRAM_ID> \
  --program-so <NEW_QUOTING_LOGIC_BINARY> \
  --reroute-order-flow router
```

Without `--parallel`, `--start-slot` must be a `Start Slot` that `sim ranges` lists. See [Select Time Window](../replaying-the-market.md#select-time-window).

The router quotes each swap at its [pricing point](README.md#interpreting-the-numbers). The new route is then simulated just before the original swap and discarded. The original transactions still run as they did onchain, so rerouting never changes the replay, and new routes never see each other's effects.
{% endstep %}

{% step %}
### Select Aggregators

The simulation can match against swaps from the major aggregators: Jupiter, DFlow, OKX, and Titan.

By default, only Jupiter transactions are rerouted, since the router currently runs Jupiter Metis and that gives the most faithful replay. Swaps from the other aggregators can be rerouted through the router as well. Their new route is always a route from the session's router.

```bash
sim run ...
    --reroute-aggregators jupiter,okx
```

For example, if a market maker isn't integrated with DFlow or Titan and can't win flow from them on mainnet, they can choose to enable only Jupiter and OKX in the simulation.
{% endstep %}

{% step %}
### Toggle Arbitrage

A meaningful portion of aggregator volume is toxic flow, especially atomic arbitrage: a circular loop like `SOL` → `USDC` → `SOL` within a single transaction.

Most atomic arbs are built by searchers outside the aggregator and only wrapped in the aggregator's instructions. That means rerouting them through the router often won't reflect how they were actually constructed.

By default, atomic arbs aren't rerouted, and every swap in a transaction that holds one counts as `filtered`. To reroute them, add the flag below. The router then quotes each arb leg by leg. An arb that can't be split into legs counts as `reroute-fail`, with an `error_category` that starts with `cycle_`.

```bash
sim run ...
    --reroute-circular-arbs
```
{% endstep %}
{% endstepper %}

{% hint style="info" %}
### Customize Conditions

Besides a new program binary (`--program-so`), `sim run` can replace account states from a file with `--account-overrides <FILE>`, for example a pool vault's deployed capital or a new pool. Add `--reroute-extra-markets <POOL>` so the router also routes through a pool it doesn't already know, such as one created in the overrides file. The file's format is in [File Formats](../../cli/file-formats.md).

To test other parameters, such as an oracle update's fair value or position in a block, use the [Rust client](../../overview/installation/rust-reference.md). The [`counterfactual_flow`](https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_flow) example changes a venue's state and measures the taker flow it wins or loses.
{% endhint %}

## Reading the Results

When the run ends, `sim` prints the reroute summary and where it saved the results:

```
↻ Reroute: 1204 detected → 1093 rerouted → 1093 simulated → 1050 succeeded | 98 filtered | 13 reroute-fail
↻ Reroute tables: s3://data.termina.technology/userdata/<your_key_name>/datasets/reroute/<run_name>.sqlite (published once the session shuts down, kept 7 days)
```

The counters count original swaps:

* `detected`: swaps in transactions that hold a swap from an aggregator in `--reroute-aggregators`.
* `filtered`: left out before quoting: atomic arbs without `--reroute-circular-arbs`, and swaps that share a transaction with an aggregator not in `--reroute-aggregators`.
* `rerouted`: the router returned a new route.
* `reroute-fail`: the router couldn't route the swap, so nothing was simulated for it.
* `simulated`, `succeeded`: new routes that were simulated, and those that didn't revert.

With `--parallel`, each session prints its own summary and saves its own file.

Swaps whose original transaction failed onchain aren't rerouted or counted. To reroute them too, add `--detect-failed-l1-swaps true`. They still aren't counted on this line: find them in the SQLite file with `original_failed = 1`.

### Query the SQLite File

Load your dataset credentials and download the file the run printed:

```bash
eval "$(sim datasets credentials)"   # fish: sim datasets credentials | source
aws s3 cp s3://data.termina.technology/userdata/<your_key_name>/datasets/reroute/<run_name>.sqlite .
sqlite3 <run_name>.sqlite
```

`reroute_quotes` has one row per detected swap. The columns used below:

* `outcome`: `rerouted`, `requote_failed` or `skipped`. `error_category` says why a swap wasn't rerouted.
* `output_bps`: the router's quoted output against the output the original swap's instruction quoted, in basis points, for `ExactIn` swaps. Positive means the router quoted more.
* `route_plan`: the new route as a JSON array of hops. Each hop's `swapInfo` names the venue (`label`) and pool (`ammKey`).
* `exec_status`: what the new transaction did: `ok`, `reverted`, `sim_error` or `skipped_missing_token_account`.

How often the new route went through your pool, and the mean quote change on those swaps:

```sql
SELECT COUNT(*)         AS routed_through_pool,
       AVG(output_bps)  AS mean_output_bps
FROM reroute_quotes
WHERE outcome = 'rerouted'
  AND EXISTS (
    SELECT 1 FROM json_each(reroute_quotes.route_plan) AS hop
    WHERE json_extract(hop.value, '$.swapInfo.ammKey') = '<POOL>'
  );
```

Why swaps weren't rerouted:

```sql
SELECT outcome, error_category, COUNT(*) AS swaps
FROM reroute_quotes
WHERE outcome != 'rerouted'
GROUP BY outcome, error_category
ORDER BY swaps DESC;
```

To compare fills rather than quotes, use `realized_output_amount` against `original_realized_output_amount`. What each number measures is in [Interpreting the Numbers](README.md#interpreting-the-numbers), and every table and column in [Output Format](../../cli/output-format.md#reroute-results).
