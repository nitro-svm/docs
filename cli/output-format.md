# Output Format

What `sim` writes, for `sim` 0.22:

* [The run's output file](#run-output-file): newline-delimited JSON from `sim run`.
* [`sim compare --json`](#sim-compare-json): one JSON object, or one array for a section.
* [Reroute results](#reroute-results): the end-of-run summary and a SQLite file, from `sim run --reroute-order-flow`.

Addresses and signatures are base58 strings. Amounts are integers in base units: lamports for SOL, the mint's smallest unit for tokens.

## Run Output File

`sim run` writes newline-delimited JSON: one JSON object per line, each with a `type` and a `data` field. Name the file with `--output-file`. The default is `<start_slot>_<end_slot>_baseline.ndjson`, or `<start_slot>_<end_slot>_override.ndjson` when `--program-so` or `--account-overrides` is set.

```json
{"type":"metadata","data":{"start_slot":400000000,"end_slot":400001000,"program_ids":["<PROGRAM_ID>"],"ran_at_unix_secs":1780023967}}
{"type":"session_started","data":{"session_id":"backtest_<uuid>","session_name":"<run_name>","start_slot":400000000,"end_slot":400001000}}
{"type":"tx","data":{"slot":400000042,"timestamp":1780020611,"signature":"<SIGNATURE>","success":true,"compute_units":84213,"logs":["Program <PROGRAM_ID> invoke [1]","..."],"sol_changes":[{"pubkey":"<WALLET>","pre_lamports":29455253127,"post_lamports":29450253127}],"token_changes":[{"pubkey":"<TOKEN_ACCOUNT>","mint":"<MINT>","owner":"<WALLET>","pre_amount":1250000000,"post_amount":1310500000,"decimals":6}]}}
{"type":"tx","data":{"slot":400000057,"timestamp":1780020617,"signature":"<SIGNATURE>","success":false,"error":"Error processing Instruction 2: custom program error: 0x1771","compute_units":41007,"logs":["..."],"sol_changes":[{"pubkey":"<WALLET>","pre_lamports":29450253127,"post_lamports":29450248127}],"token_changes":[]}}
{"type":"summary","data":{"total_transactions":847,"successes":831,"failures":16,"session_name":"<run_name>","session_ids":["backtest_<uuid>"]}}
```

### Line Order

1. One `metadata` line.
2. `session_started`, `tx` and `diff` lines as they arrive. A `--parallel` run writes one file, with one `session_started` per session and lines not sorted across sessions.
3. One `summary` line. A run stopped before it finished has no `summary`: `sim summarize` warns that its totals are partial, and `sim compare` rejects the file.

`tx` lines cover only the transactions of the programs passed with `--program-id`. Without `--program-id`, the file has no `tx` lines.

### Line Types

Fields marked optional are left out when they have no value.

| `type` | `data` fields |
| --- | --- |
| `metadata` | `start_slot`, `end_slot`, `program_ids` (optional), `program_so` (optional, the `--program-so` paths), `ran_at_unix_secs` |
| `session_started` | `session_id`, `session_name` (optional), `task_id` (optional), `start_slot`, `end_slot` |
| `tx` | `slot`, `timestamp` (optional, block time in Unix seconds), `signature`, `success`, `error` (optional), `compute_units` (optional), `logs` (array of strings), `sol_changes`, `token_changes` |
| `diff` | `slot`, `tx_hash`, `account`, `tx_index` (optional), `block_time` (optional), `pre_lamports`, `post_lamports`, `pre_state`, `post_state`, `pre_tokens`, `post_tokens` (all optional) |
| `summary` | `total_transactions`, `successes`, `failures`, `session_name` (optional), `session_ids` (optional), `diagnostics` (optional), `divergent_signatures` (optional) |

* `success`, `error`: the transaction's outcome in the replay, not onchain. Without overrides the replay matches onchain, so a failed transaction usually failed onchain too. In `custom program error: 0x1771`, `0x1771` is the program's own error code in hex (6001): look it up in the program's error list or IDL.
* `compute_units`: the compute units the transaction consumed in the replay.
* `sol_changes`: `[{pubkey, pre_lamports, post_lamports}]`, only for accounts whose balance changed. Changes include the fee, so a failed transaction still lists its fee payer.
* `token_changes`: `[{pubkey, mint, owner, pre_amount, post_amount, decimals}]`, only for token accounts whose balance changed. `owner` is `""` when unknown.
* `pre_state` / `post_state`: the account as Solana RPC account JSON (`UiAccount`). `pre_state` is absent for a created account, `post_state` for a deleted one.
* `pre_tokens` / `post_tokens`: `{mint, owner, amount, decimals}`, for token accounts.
* `diagnostics`: `{received, dedup_swallowed, malformed_dropped, reconnects}`, delivery counters for tracking down missing transactions.
* `divergent_signatures`: with `sim run --fail-on-diff`, the transactions whose replay differed from their onchain outcome, log-only differences included. At most 100 per session.

`diff` lines appear only with `--subscription account-diff`: one line per account a transaction changed.

### Reading the File

`sim summarize` prints the totals, and the balances of the accounts you pass with `--accounts`. For anything else, use [`jq`](https://jqlang.org/download/).

List the failed transactions and their errors:

```bash
jq -c 'select(.type == "tx" and (.data.success | not)) | .data | {signature, error}' baseline.ndjson
```

Count the failures by error, without the `Error processing Instruction N:` prefix, so the same error from different instructions counts once:

```bash
jq -s '[.[] | select(.type == "tx" and (.data.success | not))
    | .data.error | sub("^Error processing Instruction [0-9]+: "; "")]
  | group_by(.) | map({error: .[0], count: length}) | sort_by(-.count)' baseline.ndjson
```

List the 10 accounts whose SOL balance changed most over the run:

```bash
jq -s '[.[] | select(.type == "tx") | .data.sol_changes[]]
  | group_by(.pubkey)
  | map({pubkey: .[0].pubkey, net_lamports: (map(.post_lamports - .pre_lamports) | add)})
  | sort_by(if .net_lamports < 0 then -.net_lamports else .net_lamports end)
  | reverse | .[:10]' baseline.ndjson
```

## `sim compare --json` <a href="#sim-compare-json" id="sim-compare-json"></a>

`sim compare` matches the transactions of the baseline and the experiment by signature. Without a section, `sim compare --json` prints one object:

| Field | Contents |
| --- | --- |
| `baseline`, `experiment` | The two file paths. |
| `baseline_transactions`, `experiment_transactions`, `common_transactions` | Transaction counts. |
| `regressions` | `[{slot, signature, error}]`: success in the baseline, failure in the experiment. |
| `improvements` | `[{slot, signature, was}]`: failure in the baseline, success in the experiment. `was` is the baseline error. |
| `log_diffs` | `[{slot, signature, baseline_log_count, experiment_log_count}]`: same outcome, different number of log lines. |
| `compute_unit_increases` | `[{slot, signature, baseline_compute_units, experiment_compute_units}]`: successful in both, with compute units more than `--compute-units-threshold` percent (default 10) higher in the experiment. Largest increase first. |
| `missing` | Signatures in the baseline but not in the experiment. |
| `balance_diffs` | `[{slot, signature, sol, tokens}]`: successful in both, with a different PnL. `sol` is `[{pubkey, baseline_delta, experiment_delta}]`, `tokens` is `[{pubkey, mint, symbol, decimals, baseline_delta, experiment_delta}]`. |
| `summary` | `{regressions, improvements, log_diffs, compute_unit_increases, missing, balance_diffs}`: the count of each. |

With a section (`regressions`, `improvements`, `logs`, `compute-units` or `balances`), it prints that section's array only.

* Both files must come from finished runs, with a `summary` line, hold every transaction their `summary` line counts, and share at least one transaction. Otherwise `sim compare` exits with status 1.
* `--fail-on-diff` exits with status 1 when the selected section has an entry, or, without a section, when any section does. A transaction missing from the experiment always counts.
* Not compared: transactions only in the experiment, the contents of log lines, and `diff` lines. A transaction without `compute_units`, from a file an older `sim` wrote, is left out of `compute_unit_increases`.
* Token symbols are looked up on Solana mainnet's public RPC; a failed lookup shows `TOK`.

To gate CI on the result, use `sim compare --fail-on-diff` as in [Use in CI](README.md#use-in-ci), or check the JSON yourself:

```bash
sim compare --json baseline.ndjson experiment.ndjson \
  | jq -e '.summary.regressions == 0 and .summary.missing == 0'
```

## Reroute Results

A run with `--reroute-order-flow` prints a summary when it ends and saves its results as a SQLite file. With `--parallel`, each session prints its own summary and saves its own file. What each number compares, and where the replay was when it was simulated, is in [Interpreting the Numbers](../flows/rerouting-historical-flow/README.md#interpreting-the-numbers).

### End-of-Run Summary

```
↻ Reroute: 1204 detected → 1093 rerouted → 1093 simulated → 1050 succeeded | 98 filtered | 13 reroute-fail
◆ Venue probe: my-pool/usdc-to-token: 412 matched → filled 398, reverted_<reason> 9, unattributed 5 | 398 scored, +3.2 bps
◆ Venue probe: total: 412 matched → filled 398, reverted_<reason> 9, unattributed 5 | 398 scored, +3.2 bps
↻ Reroute tables: s3://data.termina.technology/userdata/<your_key_name>/datasets/reroute/<run_name>.sqlite (published once the session shuts down, kept 7 days)
```

The `Reroute` line counts original swaps:

| Counter | Meaning |
| --- | --- |
| `detected` | Swaps found in the slot range from the aggregators you selected. |
| `filtered` | Swaps left out before quoting, such as atomic arbitrage without `--reroute-circular-arbs`. |
| `rerouted` | Swaps the router routed. |
| `reroute-fail` | Swaps the router could not route. Nothing was simulated for them. |
| `simulated` | Rerouted swaps whose new transaction was simulated. |
| `succeeded` | Simulated swaps whose new transaction didn't revert. |

Each `Venue probe` line covers one template from `--venue-templates`, then the total:

* `matched`: hops of original swaps the template matched. Each one is probed once per matching template.
* The outcomes: `filled`, `reverted_<reason>`, `unattributed` (ran, but the measured token account was gone or unreadable afterwards) and `unsimulatable` (could not be built or run).
* `scored` and the mean bps: the change against the original swap's onchain fill, averaged with every hop counting once. Positive means the template beat the original swap. `ExactIn` templates compare the output received; `ExactOut` templates compare the input charged.

### Download the SQLite File

The file is published to your datasets when the session shuts down, and kept for 7 days. It's named after the run: `<run_name>.sqlite`, or `<run_name>/<start_slot>-<end_slot>.sqlite` for each session of a `--parallel` run. Copy the URI the run printed:

```bash
eval "$(sim datasets credentials)"   # fish: sim datasets credentials | source
aws s3 cp s3://data.termina.technology/userdata/<your_key_name>/datasets/reroute/<run_name>.sqlite .
sqlite3 <run_name>.sqlite
```

To query a `--parallel` run as one file, download its sessions' files and merge them:

```bash
aws s3 cp --recursive s3://data.termina.technology/userdata/<your_key_name>/datasets/reroute/<run_name>/ <run_name>/
for f in <run_name>/*.sqlite; do
  sqlite3 merged.sqlite "ATTACH '$f' AS s; $(sqlite3 "$f" "SELECT group_concat(
    'CREATE TABLE IF NOT EXISTS ' || name || ' AS SELECT * FROM s.' || name || ' WHERE 0; '
    || 'INSERT INTO ' || name || ' SELECT * FROM s.' || name || ';', ' ')
    FROM sqlite_master WHERE type = 'table'")"
done
```

`merged.sqlite` has every table with the rows of every session. The sessions cover different slots, so no swap appears twice.

### Tables

Every table names an original swap by `(original_signature, ix_index)`: the transaction's signature and the swap's top-level instruction index. A hop adds `leg_index`.

* Flags are `0` or `1`.
* Basis-point columns are clamped to ±100,000 (±1,000%), so one bad reading can't dominate an average. Add `WHERE ABS(<column>) < 100000` to leave clamped rows out.
* `l1` and columns whose names start with `l1_` mean the original swap as it ran onchain.

| Table | One row per | Written by |
| --- | --- | --- |
| `original_swaps` | Original swap | Both modes |
| `original_legs` | Hop of an original swap's route | Both modes |
| `original_sims` | Original swap, replayed at its pricing point | Both modes |
| `reroute_quotes` | Original swap offered to the router | Rerouting via an aggregator |
| `direct_fill_probes` | Template probe of one hop | Rerouting directly to a venue |
| `reroute_run_metrics` | Session (one row) | Both modes |
| `direct_fill_run_metrics` | Template | Rerouting directly to a venue |

**`original_swaps`**

* `slot`, `batch_index`: where the original swap executed.
* `original_failed`: the original transaction failed onchain.
* `quote_slot`, `quote_batch`, `quote_lag_blocks`: the swap's [pricing point](../flows/rerouting-historical-flow/README.md#interpreting-the-numbers), and how many blocks before it executed.

**`original_legs`**

* `source`: `l1` for the route as it ran onchain, `replay` for the route the replay produced.
* `leg_index`, `hop_index`, `share_index`: the leg's position. `leg_index` counts every share of every hop in order; a hop split across pools has one share per pool. One row per leg and `source`: `(original_signature, ix_index, source, leg_index)`.
* `program`, `label`: the venue's program and name.
* `input_mint`, `input_amount`, `output_mint`, `output_amount`.

**`original_sims`**

* `outcome`: `filled`, `reverted`, `unattributed` or `unsimulatable`. `category` is the revert reason.
* `compute_units`, `fee`.
* `simulated_output_amount`: what the original swap received, replayed at its pricing point. This is the same state the venue probes see.
* `l1_fill`: what the original swap received onchain.
* `change_vs_l1_bps`: `simulated_output_amount` against `l1_fill`, in basis points: how much the state moved between the pricing point and the swap's execution.

**`reroute_quotes`**

* `slot`, `input_mint`, `output_mint`, `amount`, `swap_mode` (`ExactIn` or `ExactOut`), `aggregator` (`jupiter`, `okx`, `titan` or `dflow`).
* `outcome`: `rerouted`, `requote_failed` (the router could not route it) or `skipped` (never offered to the router). `error_category` and `error_detail` say why.
* `quoted_output_amount`: the output the original swap's own instruction quoted: Jupiter's `quoted_out_amount` (`out_amount` for `ExactOut`), OKX's `expect_amount_out`, Titan's `expected_amount_out` or DFlow's `quoted_out_amount`. Null when the instruction has none.
* `new_output_amount`: the output the router quoted.
* `output_bps`: `new_output_amount` against `quoted_output_amount`, in basis points, for `ExactIn` swaps. Positive means the router quoted more.
* `new_route`, `route_plan`: the new route, as text and as JSON.
* `route_match`: how the new route compares with the original: `identical_pools`, `same_venues`, `pools_unknown`, `same_path`, `changed` or `unknown`.
* `exec_status`: what the new transaction did when simulated: `ok`, `reverted`, `sim_error` or `skipped_missing_token_account`. `exec_error_category` and `exec_error_detail` say why.
* `realized_output_amount`: what the new transaction received. `original_realized_output_amount`: what the original swap received onchain. Both are set only for single-swap transactions. To ask how the new route would have filled, compare these two, not the quotes.
* `original_failed`, `l1_failure_class` (`slippage`, `unrelated` or `excluded`): the original transaction failed onchain, and why. Failed originals are rerouted only with `--detect-failed-l1-swaps`.
* `slippage_bps`: the slippage the new quote allowed.

**`direct_fill_probes`**

* `template`: the template's `label`.
* `original_signature`, `ix_index`, `leg_index`: the hop probed. Its mints, amounts, `hop_index` and `share_index` are on the `original_legs` row with the same three values and `source = 'l1'`.
* `swap_mode`: `ExactIn` or `ExactOut`.
* `template_fill`: what the template filled: output for `ExactIn`, input for `ExactOut`.
* `l1_fill`: what the original swap's hop filled onchain, measured the same way.
* `change_vs_l1_bps`: `template_fill` against `l1_fill`, in basis points. Positive means the template beat the original swap.
* `outcome`: `filled`, `reverted`, `unattributed` or `unsimulatable`. `category` is the revert reason.

**`reroute_run_metrics`** holds the session's counters, such as `swaps_rerouted`, `swaps_filtered`, `requote_failures` and `swaps_simulated`, and `failure_categories`: a JSON object of failure counts by category.

**`direct_fill_run_metrics`** holds one row per template: `label`, `matched`, `outcomes` (a JSON object of probe counts by outcome), `scored` and `bps_total`. Divide `bps_total` by `scored` for the mean.

### Example Queries

Win rate, and mean basis points with every hop counting once:

```sql
SELECT template,
       COUNT(*)                   AS scored,
       AVG(change_vs_l1_bps > 0)  AS win_rate,
       AVG(change_vs_l1_bps)      AS mean_bps
FROM direct_fill_probes
WHERE outcome = 'filled' AND change_vs_l1_bps IS NOT NULL
GROUP BY template
ORDER BY scored DESC;
```

For the mean weighted by size, and for comparisons on the same state, see [Interpreting the Numbers](../flows/rerouting-historical-flow/README.md#interpreting-the-numbers).

How often the new route beat the original swap, per aggregator:

```sql
SELECT aggregator,
       COUNT(*) AS compared,
       AVG(realized_output_amount > original_realized_output_amount) AS beat_original
FROM reroute_quotes
WHERE exec_status = 'ok'
  AND swap_mode = 'ExactIn'
  AND realized_output_amount IS NOT NULL
  AND original_realized_output_amount IS NOT NULL
GROUP BY aggregator;
```
