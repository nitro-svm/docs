# CLI

`sim` runs sessions over historical Solana slots from the command line. Replay a slot range, swap in a new program binary or account overrides, reroute historical swaps, and compare the results with the onchain outcome or with another run.

This page covers `sim` 0.22. To install it, see [Installation](../overview/installation/README.md). For every flag, see the [CLI Reference](reference.md) or run `sim <command> --help`.

## Check Your API Key

```bash
export SIMULATOR_API_KEY=<API_KEY>
sim usage
```

`sim usage` prints your key's usage. A rejected key prints an `HTTP 401` error and exits with status 1. Without a key, `sim` stops before reaching the server, with `the following required arguments were not provided: --api-key <API_KEY>` and status 2.

## Commands

| Command | What it does | Guide |
| --- | --- | --- |
| `sim ranges` | Lists the slot ranges the server can replay. | [Replaying the Market](../flows/replaying-the-market.md#select-time-window) |
| `sim run` | Runs a slot range and writes the program's transactions to a file. | [Replaying the Market](../flows/replaying-the-market.md#run-a-baseline), [Rerouting Historical Flow](../flows/rerouting-historical-flow/README.md) |
| `sim summarize` | Prints totals for an output file, and balances for the accounts you name. | [Replaying the Market](../flows/replaying-the-market.md#run-a-baseline) |
| `sim compare` | Compares a baseline and an experiment: regressions, improvements, log differences, compute-unit increases, missing transactions and balance differences. | [Replaying the Market](../flows/replaying-the-market.md#compare-results) |
| `sim bundle build`, `status`, `list`, `delete` | Builds and manages your custom bundles. | [Replaying the Market](../flows/replaying-the-market.md#select-time-window) |
| `sim depth measure`, `status`, `coverage` | Requests and tracks custom depth datasets. | [Custom Datasets](../flows/generating-datasets/custom-datasets.md) |
| `sim datasets credentials` | Prints temporary, read-only S3 credentials for the prebuilt datasets and your custom datasets. | [Prebuilt Datasets](../flows/generating-datasets/prebuilt-datasets.md#access) |
| `sim usage` | Shows your API key's usage. Add `--session-details` to list each session. | [Usage and Limits](#usage-and-limits) |
| `sim update` | Replaces `sim` with the latest prebuilt binary. | [Installation](../overview/installation/README.md) |

`sim bundle build` and `sim depth measure` return a job ID. Pass it to `sim bundle status` or `sim depth status`, or add `--wait` to wait for the job to finish. With `--wait`, a job that fails or needs investigation exits with status 1.

When the server is busy, `sim run` waits up to 15 minutes for capacity, then fails.

## Output

* `sim run` writes newline-delimited JSON. See [Output Format](output-format.md).
* `sim compare --json` prints one JSON object. See [Output Format](output-format.md#sim-compare-json).
* A rerouting run also saves its results as a SQLite file. See [Output Format](output-format.md#reroute-results).
* `--account-overrides` and `--venue-templates` read JSON files. See [File Formats](file-formats.md).

## Run Names

Every `sim run` has a name, printed to stderr as the run starts. Pick one with `--name my-run`, or let the server generate a three-word one.

* 1-100 letters, digits, `-`, `_` or `.`, starting and ending with a letter or digit. Letters are lowercased.
* Unique across your runs. A name can't be reused once a run has started; a run that fails before it starts frees its name.
* A `--parallel` run has one name, shared by all of its sessions.

The name appears as `session_name` in the output file, in the `sim usage --session-details` table, and in the file name of the run's reroute results.

## Environment Variables

Every flag below can also be set through its environment variable. A flag on the command line wins over the variable.

| Variable | Flag | Commands |
| --- | --- | --- |
| `SIMULATOR_API_KEY` | `--api-key` | all but `compare`, `summarize` and `update` |
| `SIMULATOR_URL` | `--url` | all but `compare`, `summarize` and `update` |
| `SIMULATOR_START_SLOT` | `--start-slot` | `run`, `bundle build`, `bundle delete` |
| `SIMULATOR_END_SLOT` | `--end-slot` | `run`, `bundle build`, `bundle delete` |
| `SIMULATOR_ADVANCE_COUNT` | `--advance-count` | `run` |
| `SIMULATOR_PARALLEL` | `--parallel` | `run` |
| `SIMULATOR_FAIL_ON_DIFF` | `--fail-on-diff` | `run` |
| `SIMULATOR_PROGRAM_ID` | `--program-id` | `run` |
| `SIMULATOR_PROGRAM_SO` | `--program-so` | `run` |
| `SIMULATOR_ACCOUNT_OVERRIDES` | `--account-overrides` | `run` |
| `SIMULATOR_OUTPUT_FILE` | `--output-file` | `run` |
| `SIMULATOR_EXTRA_COMPUTE_UNITS` | `--extra-compute-units` | `run` |
| `SIMULATOR_SUBSCRIPTION` | `--subscription` | `run` |
| `SIMULATOR_PLAN` | `--plan` | `run` |
| `SIMULATOR_REROUTE_ORDER_FLOW` | `--reroute-order-flow` | `run` |
| `SIMULATOR_DETECT_FAILED_L1_SWAPS` | `--detect-failed-l1-swaps` | `run` |
| `SIMULATOR_REROUTE_AGGREGATORS` | `--reroute-aggregators` | `run` |
| `SIMULATOR_REROUTE_CIRCULAR_ARBS` | `--reroute-circular-arbs` | `run` |
| `SIMULATOR_REROUTE_EXTRA_MARKETS` | `--reroute-extra-markets` | `run` |
| `SIMULATOR_VENUE_TEMPLATES` | `--venue-templates` | `run` |
| `SIMULATOR_REPLAY_ACCOUNT_STATE` | `--replay-account-state` | `run` |

## Logging

`RUST_LOG` sets the log level for every command, for example `RUST_LOG=info`. Logs go to stderr. With `sim run`, add `--verbose` to see logs instead of the progress bar.

```bash
RUST_LOG=info sim run --verbose --start-slot <START> --end-slot <END> --program-id <PROGRAM_ID>
```

## Exit Status

| Status | Meaning |
| --- | --- |
| `0` | Success. |
| `1` | Error, or `--fail-on-diff` found a difference. Also: `sim run` when no `--program-id` transaction ran in the range (`no matching transactions found in the requested slot range`); `sim compare` when either file has no `summary` line or a different number of transaction lines than it counts, or the files share no transactions; `sim bundle build --wait` and `sim depth measure --wait` when the job failed or needs investigation. |
| `2` | Usage error: unknown flag, missing or invalid value, including a missing API key. |
| `130` | `sim run` was stopped by Ctrl+C, SIGTERM or SIGQUIT before it finished. The output file is partial. |

`--fail-on-diff` means different things on the two commands:

* `sim run --fail-on-diff` checks the simulated outcome of each transaction against its onchain outcome, and names the transactions that differed in the `summary` line's `divergent_signatures`. Use it to check that a replay is faithful, not with `--program-so`: a new program binary is expected to differ from onchain.
* `sim compare --fail-on-diff` checks an experiment against a baseline. Use it to gate a new program binary.

## Stopping a Run

* First Ctrl+C: `sim run` closes its sessions, writes what it recorded to the output file and exits with status 130. The file has no `summary` line, so `sim compare` rejects it.
* Second Ctrl+C: exits right away with status 130, without waiting for the sessions to close.

SIGTERM and SIGQUIT behave the same way.

## Use in CI

Pin the version, set `SIMULATOR_API_KEY` from your CI secrets, and gate a new program binary on regressions and compute-unit increases:

```bash
set -e
cargo install sim-cli --locked --version 0.22.0

sim run --start-slot <START_SLOT> --end-slot <END_SLOT> --program-id <PROGRAM_ID> \
  --output-file baseline.ndjson
sim run --start-slot <START_SLOT> --end-slot <END_SLOT> --program-id <PROGRAM_ID> \
  --program-so ./target/deploy/program.so --output-file experiment.ndjson

sim compare baseline.ndjson experiment.ndjson regressions --fail-on-diff
sim compare baseline.ndjson experiment.ndjson compute-units --fail-on-diff \
  --compute-units-threshold 5
```

The job fails when:

* either `sim run` exits non-zero, including `130` when a signal stopped it;
* either file has no `summary` line, because its run didn't finish;
* either file has a different number of transaction lines than its `summary` line counts;
* the two files share no transactions, for example because of different slots or `--program-id`;
* a transaction from the baseline is missing from the experiment;
* the section has an entry: a success that became a failure, or a transaction whose compute units rose by more than 5%.

Compute units are compared only for transactions that succeeded in both runs. The default threshold is 10%. Drop the section to fail on any difference, PnL included.

Before you rely on the gate:

* Run the experiment once with the deployed program's own binary. `sim compare` should find no differences.
* Run the baseline and the experiment in the same job, so both use the same server.
* `--extra-compute-units` raises the limit only of transactions that set one with `SetComputeUnitLimit`. Pass the same value to both runs, so only the program differs.

### Choosing a Range

Prebuilt ranges cover the past 30 days, so a range you hard-code stops working once it rolls off. Either:

* Pick the newest range on every run, and stay inside it:

  ```bash
  RANGE=$(sim ranges --kind transactions --json | jq -c 'max_by(.bundle_start_slot)')
  START_SLOT=$(jq '.bundle_start_slot' <<<"$RANGE")
  END_SLOT=$(jq '[.bundle_start_slot + 1000, .max_bundle_end_slot // infinite] | min' <<<"$RANGE")
  ```

* Or build a custom bundle once with `sim bundle build`, and pin the job to its range. Custom bundles aren't removed on a schedule; `sim bundle delete` removes one.

Your program must run in the range: a run with no `--program-id` transactions exits with status 1. Something not working? See [Troubleshooting](../troubleshooting.md).

## Usage and Limits

`sim usage` shows the last 30 days by default. Pass `--since` and `--until` for another window.

```
Usage Report
  API key:      acme
  Time range:   2026-09-27 00:00:00 UTC to 2026-10-27 12:00:00 UTC

Sessions
  Completed:  14
  Incomplete: 1

Compute
  Slots executed:         61200
  Total session duration: 2h 41m

Daily (UTC)
  Day           Started  Completed  Incomplete          Slots     Duration       Cost
  2026-10-24          6          6           0          24000        1h 2m     $24.00
  2026-10-26          9          8           1          37200       1h 39m     $37.20
```

* A session is billed per executed slot. A session that averaged more than a set wall time per slot is billed per minute instead. The `Cost` column prices each day at your key's rates.
* Custom bundle builds and depth jobs share one request limit per key. Over it, a request gets `HTTP 429`: wait and send it again. A bundle build resent with the same `--idempotency-key`, or a depth request identical to an earlier one, doesn't count again.
* Add `--plan` to `sim run` to see how a range splits into sessions, and an estimated runtime, before you run it.
