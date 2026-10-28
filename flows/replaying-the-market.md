# Replaying the Market

{% stepper %}
{% step %}
### Select Time Window

Check if the range matches any prebuilt bundles, which cover the past 30 days. If needed, filter the search with `--after <DATE>` or `--before <DATE>`.

```bash
sim ranges
```

```
Start Time (UTC)               End Time (UTC)                 Start Slot      End Slot        Bundle Size     Kind            Scope
----------------------------------------------------------------------------------------------------------------------------------
2026-06-30T17:53:10Z           2026-06-30T19:00:53Z           429926541       429936540       10000           transactions    global
2026-06-30T18:23:38Z           2026-06-30T18:27:02Z           429931041       429931540       500             transactions    global
2026-06-30T17:53:10Z           2026-06-30T19:00:53Z           429926541       429936540       10000           account_state   user
```

* `sim run` replays `transactions` ranges. Account-state ranges are only for `sim run --replay-account-state`. List them with `sim ranges --kind account-state`; the `Kind` column and the API spell the kind `account_state`.
* A run starts at a listed `Start Slot` and can end at any slot up to that row's `End Slot`. With `--parallel`, `--start-slot` may fall inside a listed range, but the first session then starts at that range's `Start Slot`.
* For a longer range, add `--parallel` to split it across concurrent sessions.
* Add `--plan` to `sim run` to print how the range splits into sessions and an estimated runtime, without running it.
* `unbounded` in End Slot means the server reported no end for the range.
* For a range that stays valid in CI, see [Choosing a Range](../cli/README.md#choosing-a-range).

For ranges older than 30 days or ones that fall between prebuilt bundles, build a custom one instead. Add `--wait` to wait for the job to finish, or follow it with `sim bundle status <JOB_ID>`.

```bash
sim bundle build --start-slot 400000000 --end-slot 400001000 --wait
```
{% endstep %}

{% step %}
### Run a Baseline

Start a session to replay transactions and capture the existing behavior.

```bash
# Baseline
sim run \
  --start-slot 429926541 \
  --end-slot 429927540 \
  --program-id <PROGRAM_ID> \
  --output-file baseline.ndjson
```

`--program-id` selects the program whose transactions are recorded. Without it, the run records no transactions. Pass the same `--program-id` to the baseline and the experiment. If the program ran no transaction in the range, the run ends with `no matching transactions found in the requested slot range` and exit status 1.

When the run ends, `sim` prints the totals:

```
✔ Simulation complete
  Transactions : 847
  Successes    : 831
  Failures     : 16
  Output file  : baseline.ndjson
```

The output file is newline-delimited JSON: one JSON object per line, each with a `type` (`metadata`, `session_started`, `tx`, `diff` or `summary`) and its `data`. Every field is described in [Output Format](../cli/output-format.md#run-output-file).

<details>

<summary>baseline.ndjson</summary>

```json
{"type":"metadata","data":{"start_slot":429926541,"end_slot":429927540,"program_ids":["<PROGRAM_ID>"],"ran_at_unix_secs":1782897164}}
{"type":"session_started","data":{"session_id":"backtest_<uuid>","session_name":"<run_name>","start_slot":429926541,"end_slot":429927540}}
{"type":"tx","data":{"slot":429926583,"timestamp":1782842007,"signature":"<SIGNATURE>","success":true,"compute_units":84213,"logs":["Program <PROGRAM_ID> invoke [1]","..."],"sol_changes":[{"pubkey":"<WALLET>","pre_lamports":29455253127,"post_lamports":29450253127}],"token_changes":[{"pubkey":"<TOKEN_ACCOUNT>","mint":"<MINT>","owner":"<WALLET>","pre_amount":1250000000,"post_amount":1310500000,"decimals":6}]}}
{"type":"tx","data":{"slot":429926598,"timestamp":1782842013,"signature":"<SIGNATURE>","success":false,"error":"Error processing Instruction 2: custom program error: 0x1771","compute_units":41007,"logs":["..."],"sol_changes":[{"pubkey":"<WALLET>","pre_lamports":29450253127,"post_lamports":29450248127}],"token_changes":[]}}
{"type":"summary","data":{"total_transactions":847,"successes":831,"failures":16,"session_name":"<run_name>","session_ids":["backtest_<uuid>"]}}
```

</details>

`success` is the outcome in the replay: without overrides it matches onchain, so a failed transaction usually failed onchain too. `sol_changes` and `token_changes` list only the accounts whose balance changed, and SOL changes include the fee.

`sim` provides a helper that reads the output file to summarize the run.

```bash
sim summarize baseline.ndjson
```

```
File         : baseline.ndjson
Slot range   : 429926541 → 429927540
Program IDs  : <PROGRAM_ID>
Session(s)   : 1
Ran at       : 2026-07-01 09:12:44 UTC

i Transactions : 847  (831 successes, 16 failures)
```

```bash
# Trace an account's balance changes
sim summarize baseline.ndjson --accounts <WALLET>
```

```
File         : baseline.ndjson
Slot range   : 429926541 → 429927540
Program IDs  : <PROGRAM_ID>
Session(s)   : 1
Ran at       : 2026-07-01 09:12:44 UTC

i Transactions : 847  (831 successes, 16 failures)

Account: <WALLET>
  <MINT>                                              1250.000000 → 1310.500000  (Δ +60.500000)
  SOL                                                 29.455253127 → 29.293640983  (Δ -0.161612144)
```

For a file with no `summary` line, `sim summarize` warns that the totals cover only what was written before the run stopped.
{% endstep %}

{% step %}
### Run an Experiment

Build a modified program with parameter changes, feature upgrades, or bug fixes. Substitute it in place of the original binary and run the new logic against the same historical flow.

```bash
# Experiment
sim run \
  --start-slot 429926541 \
  --end-slot 429927540 \
  --program-id <PROGRAM_ID> \
  --program-so <PATH_TO_NEW_BINARY> \
  --output-file experiment.ndjson
```

To replace several programs, repeat `--program-id` and `--program-so` in the same order.

If the new program needs more compute units, raise each transaction's limit with `--extra-compute-units <EXTRA_UNITS>`. Only transactions that set a limit with `SetComputeUnitLimit` get the extra units. Pass the same value to the baseline, so only the program differs between the runs.
{% endstep %}

{% step %}
### Compare Results

Once both the baseline and experiment are complete, compare the two runs. `sim compare` matches their transactions by signature:

* regressions: succeeded in the baseline, fails in the experiment
* improvements: failed in the baseline, succeeds in the experiment
* log differences: same result, different number of log lines
* compute-unit increases: succeeded in both, with compute units more than 10% higher in the experiment (`--compute-units-threshold` sets the percent), largest increase first
* missing: in the baseline but not in the experiment
* balance differences: transactions that succeeded in both runs but changed some account's SOL or token balance by a different amount. Each lists only those accounts, as baseline change → experiment change, largest difference first

```bash
sim compare baseline.ndjson experiment.ndjson
```

```
Baseline (baseline.ndjson) versus Experiment (experiment.ndjson)
Baseline transactions: 847
Experiment transactions: 847
Common transactions: 847

✖ Regressions (success → failure): 2
  slot 429926583  4vJ9JU1b…8RB3ixfE
    error: Error processing Instruction 2: custom program error: 0x1771
  slot 429927112  2ZcRmx9F…kTw7HhQe
    error: Error processing Instruction 2: custom program error: 0x1771

~ Log differences: 1
  slot 429926901  5yQkqn2C…Fq7mWJd1  (14 baseline logs → 16 experiment logs)

▲ Compute-unit increases (more than 10%): 1
  slot 429926642  3nTPWm4s…Vb8ZQe2K  (84213 → 97540 compute units, +15.8253%)

$ Balance differences: 1
  slot 429926642  3nTPWm4s…Vb8ZQe2K
    USDC  7mR3kLpQ…Xz4vNa1D  +60.500000 → +61.120000  (+1.0248%)


Summary: 2 regressions, 0 improvements, 1 log diffs, 1 compute-unit increases, 0 missing, 1 balance diffs
```

`sim compare` exits with status 1 if either file has no `summary` line, because its run didn't finish, if a file has a different number of transaction lines than its `summary` line counts, or if the two files share no transactions: then they recorded different programs, or none, so check `--program-id`.

```bash
# Only show what broke
sim compare baseline.ndjson experiment.ndjson regressions
```

```bash
# Check PnL impact
sim compare baseline.ndjson experiment.ndjson balances
```

```bash
# Only compute units more than 5% higher
sim compare baseline.ndjson experiment.ndjson compute-units --compute-units-threshold 5
```

Add `--json` for machine-readable output. Its format, and what `sim compare` doesn't check, are in [Output Format](../cli/output-format.md#sim-compare-json).

To gate CI on the result, add `--fail-on-diff`: `sim compare` then exits with status 1 if the selected section, or any section when none is given, has an entry, or if the experiment is missing a transaction. The full recipe is in [Use in CI](../cli/README.md#use-in-ci).

```bash
# Fail if any transaction went from success to failure
sim compare baseline.ndjson experiment.ndjson regressions --fail-on-diff
```

`sim run --fail-on-diff` is a different check: it compares each simulated transaction with its onchain outcome, not with another run.
{% endstep %}
{% endstepper %}
