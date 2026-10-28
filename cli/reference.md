# CLI Reference

Every `sim` command and flag, for `sim` 0.22.

{% hint style="info" %}
This page is generated from `sim`'s help text, so it matches `sim <command> --help`. For a guided tour, start with the [CLI overview](README.md).
{% endhint %}

**Command Overview:**

* [`sim`↴](#sim)
* [`sim run`↴](#sim-run)
* [`sim compare`↴](#sim-compare)
* [`sim compare regressions`↴](#sim-compare-regressions)
* [`sim compare improvements`↴](#sim-compare-improvements)
* [`sim compare logs`↴](#sim-compare-logs)
* [`sim compare compute-units`↴](#sim-compare-compute-units)
* [`sim compare balances`↴](#sim-compare-balances)
* [`sim summarize`↴](#sim-summarize)
* [`sim update`↴](#sim-update)
* [`sim ranges`↴](#sim-ranges)
* [`sim usage`↴](#sim-usage)
* [`sim bundle`↴](#sim-bundle)
* [`sim bundle build`↴](#sim-bundle-build)
* [`sim bundle status`↴](#sim-bundle-status)
* [`sim bundle list`↴](#sim-bundle-list)
* [`sim bundle delete`↴](#sim-bundle-delete)
* [`sim depth`↴](#sim-depth)
* [`sim depth measure`↴](#sim-depth-measure)
* [`sim depth status`↴](#sim-depth-status)
* [`sim depth coverage`↴](#sim-depth-coverage)
* [`sim datasets`↴](#sim-datasets)
* [`sim datasets credentials`↴](#sim-datasets-credentials)

## `sim`

Run and compare Termina simulator backtests against historical Solana slots

**Usage:** `sim <COMMAND>`

```text
Getting started:
  export SIMULATOR_API_KEY=<your key>
  sim ranges              # slot ranges the server can replay
  sim run --start-slot <slot> --end-slot <slot> --program-id <program> \
    --output-file baseline.ndjson
  sim summarize baseline.ndjson

Set RUST_LOG (e.g. RUST_LOG=info) for detailed logs.

Exit status:
  0    Success
  1    Error, or --fail-on-diff found a difference
  2    Usage error (unknown flag, missing or invalid value)
  130  `sim run` was stopped by Ctrl+C, SIGTERM or SIGQUIT before it finished;
       the output file is partial and has no `summary` line

Documentation: https://docs.termina.technology/documentation
```

**Subcommands:**

* `run` — Run a backtest simulation over a slot range
* `compare` — Compare two simulation output files (baseline vs experiment)
* `summarize` — Extract and print a summary of a simulation output file
* `update` — Replace the sim binary with the latest build for this platform
* `ranges` — List the slot ranges the server can replay
* `usage` — Show API key usage metrics for a time window
* `bundle` — Build, inspect and delete your custom bundles
* `depth` — Request and track venue-depth datasets
* `datasets` — Get read access to the prebuilt and custom datasets



## `sim run`

Run a backtest simulation over a slot range

When the server is busy, the run waits up to 15 minutes for capacity before failing.

**Usage:** `sim run [OPTIONS] --api-key <API_KEY>`

```text
Examples:
  # Baseline run: record a program's transactions over a slot range
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> --program-id <PROGRAM_ID>

  # Override a program with a new program binary and collect its transactions
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> \
    --program-id <PROGRAM_ID> --program-so ./target/deploy/program.so

  # Split a longer range across parallel sessions
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> --parallel \
    --program-id <PROGRAM_ID>

  # Replace account states from a file before the range runs
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> \
    --program-id <PROGRAM_ID> --account-overrides ./accounts.json

  # Reroute swaps via an aggregator
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> \
    --program-id <PROGRAM_ID> --reroute-order-flow

  # Reroute swaps directly to your own venue
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> \
    --program-id <PROGRAM_ID> --reroute-order-flow venue \
    --venue-templates ./templates.json

  # Replay recorded account state (see `sim bundle build --account-state`)
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> \
    --program-id <PROGRAM_ID> --replay-account-state

  # In CI: fail if any simulated outcome differs from the onchain outcome
  sim run --start-slot <START_SLOT> --end-slot <END_SLOT> \
    --program-id <PROGRAM_ID> --fail-on-diff

Take <START_SLOT> and <END_SLOT> from `sim ranges`.
```

**Options:**

* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--start-slot <START_SLOT>` — First slot of the range (inclusive)

   Environment variable: `SIMULATOR_START_SLOT`
* `--end-slot <END_SLOT>` — Last slot of the range (inclusive)

   Environment variable: `SIMULATOR_END_SLOT`
* `--advance-count <ADVANCE_COUNT>` — Blocks to execute per step; skipped slots are not counted. Defaults to the whole range in a single step; set it lower to step through the range in smaller increments

   Environment variable: `SIMULATOR_ADVANCE_COUNT`
* `--parallel` — Split the range across up to 50 concurrent sessions. The output still goes to one file

   Environment variable: `SIMULATOR_PARALLEL`

  Default value: `false`
* `--name <NAME>` — Name for this run, unique across your runs; generated when unset
* `--fail-on-diff` — Exit with status 1 if any simulated transaction differs from its onchain outcome: a different success or failure, an execution error, or a different balance change. The `summary` line's `divergent_signatures` names them. To compare two runs instead, use `sim compare --fail-on-diff`

   Environment variable: `SIMULATOR_FAIL_ON_DIFF`

  Default value: `false`
* `--program-id <PROGRAM_ID>` — Program whose transactions to record (repeatable). Without one, the run records no transactions

   Environment variable: `SIMULATOR_PROGRAM_ID`
* `--program-so <PROGRAM_SO>` — Program binary (.so) that replaces a `--program-id` for the run. Repeat it in the same order as `--program-id`

   Environment variable: `SIMULATOR_PROGRAM_SO`
* `--account-overrides <FILE>` — JSON file of account states, keyed by address, that replace the onchain ones at `--start-slot`. Programs are refused; replace one with `--program-so`. Format: `https://docs.termina.technology/documentation/cli/file-formats#account-overrides`

   Environment variable: `SIMULATOR_ACCOUNT_OVERRIDES`
* `--output-file <OUTPUT_FILE>` — File to write the run's output to, as newline-delimited JSON. Defaults to `<start_slot>_<end_slot>_baseline.ndjson`, or `_override.ndjson` when `--program-so` or `--account-overrides` is set

   Environment variable: `SIMULATOR_OUTPUT_FILE`
* `--extra-compute-units <EXTRA_COMPUTE_UNITS>` — Extra compute units to add to each transaction's compute unit limit. Only transactions that set a limit (a `SetComputeUnitLimit` instruction) get them

   Environment variable: `SIMULATOR_EXTRA_COMPUTE_UNITS`
* `--subscription <SUBSCRIPTION>` — What to record for each `--program-id` transaction. Defaults to `transaction`

   Environment variable: `SIMULATOR_SUBSCRIPTION`

  Possible values:
  - `account-diff`:
    Transaction records plus, for each account a transaction changes, its state before and after
  - `transaction`:
    One record per transaction, with its logs and balance changes

* `--verbose` — Show logs instead of a progress bar. `RUST_LOG` sets the log level (default `info`)

  Default value: `false`
* `--plan` — Print how the run would be split into sessions and an estimated duration, then exit without running

   Environment variable: `SIMULATOR_PLAN`

  Default value: `false`
* `--reroute-order-flow <MODE>` — Reroute historical aggregator swaps and compare each new fill with the original swap.

   Each original swap is priced at its pricing point, `quote_lag_blocks` blocks before the block it executed in. `router`: the router quotes at the pricing point, and the new transaction is simulated just before the original swap, in its block. `venue`: each probe is simulated at the pricing point. Both modes also replay the original swap at the pricing point. Every simulation is discarded, so the replay is unchanged.

   The bare flag means `router`; `router,venue` runs both. When the run ends, the results are saved as a SQLite file kept for 7 days: the run prints its URI, and `sim datasets credentials` gives read access. With `--parallel`, each session prints its own results and saves its own file.

   Environment variable: `SIMULATOR_REROUTE_ORDER_FLOW`

  Possible values:
  - `router`:
    Reroute each swap through the router; the new route is simulated and discarded, and the original transaction stays in place
  - `venue`:
    Price each swap directly on a venue, using `--venue-templates`; each probe is simulated and discarded, and the original transaction stays in place

* `--detect-failed-l1-swaps <BOOL>` — Also reroute swaps whose original transaction failed on slippage or after the swap ran. They are reported separately from the other reroute results. Ignored without `--reroute-order-flow`. `l1` in the flag's name means the original swap

   Environment variable: `SIMULATOR_DETECT_FAILED_L1_SWAPS`

  Default value: `false`

  Possible values: `true`, `false`

* `--reroute-aggregators <REROUTE_AGGREGATORS>` — Aggregators whose historical swaps to reroute, comma-separated: `jupiter`, `okx`, `titan`, `dflow`. Both modes take only these swaps, so a run that reroutes directly to a venue must name every aggregator whose swaps it wants priced. Rerouting via an aggregator always builds a Jupiter route, so naming another aggregator compares its swaps with Jupiter. Ignored without `--reroute-order-flow`

   Environment variable: `SIMULATOR_REROUTE_AGGREGATORS`

  Default value: `jupiter`
* `--reroute-scopes <REROUTE_SCOPES>` — Where a swap may have run to be rerouted, comma-separated: `top-level` (an aggregator instruction of the transaction's own) and `nested` (an aggregator another instruction CPI'd, rerouted on its own for a funded stand-in taker). Ignored without `--reroute-order-flow`

   Environment variable: `SIMULATOR_REROUTE_SCOPES`

  Default value: `top-level`
* `--reroute-circular-arbs` — Also reroute atomic arbitrage (same input and output mint): the router quotes each arb leg by leg and builds it back as one circular route. An arb that can't be split into legs fails with a `cycle_*` category. Off by default: most arbs are built by searchers, so a router reroute rarely reflects how they were constructed

   Environment variable: `SIMULATOR_REROUTE_CIRCULAR_ARBS`
* `--reroute-extra-markets <POOL>` — Extra pools for the router to consider, comma-separated. Each must exist at `--start-slot`, onchain or in `--account-overrides`. Needs `--reroute-order-flow router`

   Environment variable: `SIMULATOR_REROUTE_EXTRA_MARKETS`
* `--venue-templates <FILE>` — JSON file of venue templates, each the instruction that prices a swap directly on your venue. Needs `--reroute-order-flow venue`. Format: `https://docs.termina.technology/documentation/cli/file-formats#venue-templates`

   Environment variable: `SIMULATOR_VENUE_TEMPLATES`
* `--replay-account-state` — Replay account state recorded with `sim bundle build --account-state` instead of executing the range's transactions.

   Start at an account-state bundle's `Start Slot`, and end at or before its `End Slot`. With `--parallel`, the range is split across your account-state bundles (build several with `--bundle-size`). List them with `sim ranges --kind account-state`. A run that changes execution, such as `--extra-compute-units`, is rejected.

   Environment variable: `SIMULATOR_REPLAY_ACCOUNT_STATE`

  Default value: `false`
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim compare`

Compare two simulation output files (baseline vs experiment)

**Usage:** `sim compare [OPTIONS] <BASELINE> <EXPERIMENT> [COMMAND]`

```text
Examples:
  # Run the same range with the onchain program and with a new program
  # binary, then compare.
  sim run --start-slot <slot> --end-slot <slot> --program-id <program> \
    --output-file baseline.ndjson
  sim run --start-slot <slot> --end-slot <slot> --program-id <program> \
    --program-so ./program.so --output-file experiment.ndjson
  sim compare baseline.ndjson experiment.ndjson

  # Only the transactions that went from success to failure, as JSON.
  sim compare --json baseline.ndjson experiment.ndjson regressions

  # In CI: exit with status 1 if the new program binary turned any successful
  # transaction into a failure, or the experiment is missing any transaction.
  sim compare baseline.ndjson experiment.ndjson regressions --fail-on-diff

  # In CI: exit with status 1 if any transaction's compute units rose by more
  # than 5%.
  sim compare baseline.ndjson experiment.ndjson compute-units --fail-on-diff --compute-units-threshold 5

Both files must come from finished runs (with a `summary` line), hold every
transaction their summary counts, and share at least one transaction;
otherwise compare exits with status 1.

Token symbols are looked up on Solana mainnet's public RPC; a token whose
lookup fails is shown as TOK.
```

**Subcommands:**

* `regressions` — Only show regressed transactions (success → failure)
* `improvements` — Only show improved transactions (failure → success)
* `logs` — Only show transactions with the same status but a different number of log lines
* `compute-units` — Only show transactions successful in both runs whose compute units rose by more than --compute-units-threshold percent, largest increase first
* `balances` — Only show successful transactions whose PnL differs, largest difference first

**Arguments:**

* `<BASELINE>` — Output file of the baseline run (`sim run --output-file`)
* `<EXPERIMENT>` — Output file of the experiment run to check against the baseline, e.g. one with `--program-so` or `--account-overrides`

**Options:**

* `--json` — Output results as JSON instead of human-readable text
* `--fail-on-diff` — Exit with status 1 if the selected section has any entry, or, without a section, if any section does. Transactions missing from the experiment always count. Results are still printed first
* `--compute-units-threshold <PERCENT>` — Count a transaction in the `compute-units` section when its compute units rose by more than this percent

  Default value: `10`



## `sim compare regressions`

Only show regressed transactions (success → failure)

**Usage:** `sim compare regressions`



## `sim compare improvements`

Only show improved transactions (failure → success)

**Usage:** `sim compare improvements`



## `sim compare logs`

Only show transactions with the same status but a different number of log lines

**Usage:** `sim compare logs`



## `sim compare compute-units`

Only show transactions successful in both runs whose compute units rose by more than --compute-units-threshold percent, largest increase first

**Usage:** `sim compare compute-units`



## `sim compare balances`

Only show successful transactions whose PnL differs, largest difference first

**Usage:** `sim compare balances`



## `sim summarize`

Extract and print a summary of a simulation output file

**Usage:** `sim summarize [OPTIONS] <FILE>`

```text
Examples:
  sim summarize baseline.ndjson

  # Also show opening and closing SOL and token balances for two wallets.
  sim summarize baseline.ndjson --accounts <wallet>,<wallet>
```

**Arguments:**

* `<FILE>` — Simulation output file to summarize (`sim run --output-file`)

**Options:**

* `--accounts <ACCOUNTS>` — Comma-separated list of wallet addresses to trace balances across all transactions



## `sim update`

Replace the sim binary with the latest build for this platform

**Usage:** `sim update`



## `sim ranges`

List the slot ranges the server can replay

**Usage:** `sim ranges [OPTIONS]`

```text
Examples:
  sim ranges
  sim ranges --after 2026-09-01 --before 2026-09-30

  # Ranges `sim run --replay-account-state` can replay.
  sim ranges --kind account-state

  # Ranges built for your API key, as JSON.
  sim ranges --scope user --json

Columns:
  Start Slot   pass it to `sim run --start-slot` as printed
  End Slot     the last slot a run starting at Start Slot can reach
  Bundle Size  the range's length in slots
  unbounded    the server reported no end slot for the range
```

**Options:**

* `--after <AFTER>` — Only show ranges that start at or after this point. Accepts a slot number or a timestamp (RFC 3339 or YYYY-MM-DD)
* `--api-key <API_KEY>` — Your simulator API key; needed to list the ranges built for it

   Environment variable: `SIMULATOR_API_KEY`
* `--before <BEFORE>` — Only show ranges that end at or before this point. Accepts a slot number or a timestamp (RFC 3339 or YYYY-MM-DD)
* `--kind <KIND>` — Only show ranges of this kind

  Possible values:
  - `transactions`:
    The transactions a session executes
  - `account-state`:
    The account state `run --replay-account-state` replays

* `--scope <SCOPE>` — Only show ranges with this availability

  Possible values:
  - `global`:
    Ranges of prebuilt bundles, available to everyone
  - `user`:
    Ranges of custom bundles, built for your API key

* `--size <SIZE>` — Only show ranges whose bundle size is within 10% of this value
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim usage`

Show API key usage metrics for a time window

**Usage:** `sim usage [OPTIONS] --api-key <API_KEY>`

```text
Examples:
  sim usage
  sim usage --since 2026-10-01 --until 2026-10-08 --session-details
  sim usage --since 2026-10-01T12:00:00Z --json
```

**Options:**

* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--since <SINCE>` — Report window start (RFC 3339 or YYYY-MM-DD). Defaults to the server's configured window
* `--until <UNTIL>` — Report window end (RFC 3339 or YYYY-MM-DD). Defaults to now
* `--session-details` — Include the individual sessions
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim bundle`

Build, inspect and delete your custom bundles

**Usage:** `sim bundle <COMMAND>`

**Subcommands:**

* `build` — Issue a bundle build over a slot range
* `status` — Check the status of a previously-issued bundle build
* `list` — List your bundle builds and custom bundles
* `delete` — Delete one of your custom bundles



## `sim bundle build`

Issue a bundle build over a slot range

**Usage:** `sim bundle build [OPTIONS] --api-key <API_KEY>`

```text
Examples:
  # Build slots 380000000-380099999 as ten 10,000-slot bundles, and wait.
  sim bundle build --start-slot 380000000 --end-slot 380099999 \
    --bundle-size 10000 --wait

  # Also record account-state bundles, so each bundle can be replayed by its
  # own `sim run --replay-account-state` session.
  sim bundle build --start-slot 380000000 --end-slot 380099999 \
    --bundle-size 10000 --account-state

  # Build a bundle of the slots right after the latest snapshot.
  sim bundle build --latest --bundle-size 10000

  # Follow a build submitted without --wait, or list recent builds.
  sim bundle status <job-id>
  sim bundle list --limit 10
```

**Options:**

* `--start-slot <START_SLOT>` — First slot of the range (inclusive)

   Environment variable: `SIMULATOR_START_SLOT`
* `--end-slot <END_SLOT>` — Last slot of the range (inclusive)

   Environment variable: `SIMULATOR_END_SLOT`
* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--latest` — Build forward from the latest snapshot slot; requires `--bundle-size`
* `--bundle-size <BUNDLE_SIZE>` — Slots per bundle. Omit to build the whole range as one bundle
* `--account-state` — Also record an account-state bundle over each bundle built, which `sim run --replay-account-state` can then replay instead of executing those slots. Adds a full execution pass, so it roughly doubles the build.

   A session replays one bundle, so pair this with `--bundle-size` to replay the range across parallel sessions — one per bundle. List what a build produced with `sim ranges --kind account-state`.
* `--idempotency-key <IDEMPOTENCY_KEY>` — Dedupe key: resubmitting the same key returns the existing build instead of triggering (or re-charging for) a new one
* `--wait` — Wait until the build finishes instead of returning once it is submitted. Exits with status 1 when the build fails
* `--poll-secs <POLL_SECS>` — Seconds between status polls when `--wait` is set

  Default value: `15`
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim bundle status`

Check the status of a previously-issued bundle build

**Usage:** `sim bundle status [OPTIONS] --api-key <API_KEY> <JOB_ID>`

**Arguments:**

* `<JOB_ID>` — The job ID returned by `sim bundle build`

**Options:**

* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim bundle list`

List your bundle builds and custom bundles

**Usage:** `sim bundle list [OPTIONS] --api-key <API_KEY>`

**Options:**

* `--limit <LIMIT>` — Maximum builds to return, newest first
* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim bundle delete`

Delete one of your custom bundles

**Usage:** `sim bundle delete [OPTIONS] --api-key <API_KEY>`

```text
Examples:
  # Delete a transactions bundle, asking for confirmation first.
  sim bundle delete --start-slot 380000000 --end-slot 380009999

  # Delete the matching account-state bundle without a prompt.
  sim bundle delete --start-slot 380000000 --end-slot 380009999 \
    --kind account-state --yes
```

**Options:**

* `--start-slot <START_SLOT>` — First slot of the range (inclusive)

   Environment variable: `SIMULATOR_START_SLOT`
* `--end-slot <END_SLOT>` — Last slot of the range (inclusive)

   Environment variable: `SIMULATOR_END_SLOT`
* `--kind <KIND>` — Which bundle over this range to delete

  Default value: `transactions`

  Possible values:
  - `transactions`:
    The transactions a session executes
  - `account-state`:
    The account state `run --replay-account-state` replays

* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--yes` — Delete without asking for confirmation
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim depth`

Request and track venue-depth datasets

**Usage:** `sim depth <COMMAND>`

**Subcommands:**

* `measure` — Measure venue depth over a slot range and publish it as a dataset
* `status` — Check the status of a previously-issued depth job
* `coverage` — Show which markets, venues and rungs a depth job would sample



## `sim depth measure`

Measure venue depth over a slot range and publish it as a dataset

**Usage:** `sim depth measure [OPTIONS] --start-slot <START_SLOT> --end-slot <END_SLOT> --source <SOURCE> --api-key <API_KEY>`

```text
Examples:
  # Measure onchain depth for every covered market, and wait for the job.
  sim depth measure --start-slot 380000000 --end-slot 380099999 \
    --source onchain --wait

  # Sample one market at every oracle update instead of once per slot.
  sim depth measure --start-slot 380000000 --end-slot 380099999 \
    --source onchain --granularity oracle --market SOL-USDC

  # Measure router quotes for two markets, then check on the job later.
  sim depth measure --start-slot 380000000 --end-slot 380099999 \
    --source router --market SOL-USDC --market NVDAx-USDC
  sim depth status <job-id>
```

**Options:**

* `--start-slot <START_SLOT>` — First slot to capture, inclusive. Each row is filed under the UTC date its slot falls on
* `--end-slot <END_SLOT>` — Last slot to capture, inclusive
* `--source <SOURCE>` — Where a job's depth numbers come from

  Possible values:
  - `onchain`:
    What the venues' own onchain liquidity would have filled
  - `router`:
    What the router would have quoted

* `--granularity <GRANULARITY>` — When to sample the source: once per slot, or `oracle`: after each batch where the venue's program or an oracle update wrote an account a probe reads (`onchain` only)

  Default value: `slot`

  Possible values:
  - `slot`:
    On a fixed one-slot grid. The default
  - `oracle`:
    After each batch where the venue's program or an oracle update wrote an account a probe reads. `onchain` only

* `--market <MARKETS>` — Capture only this market, by id (see `sim depth coverage`). Repeat to capture multiple markets, leave unset to capture all markets
* `--wait` — Wait until the job finishes instead of returning once it is submitted. Prints the dataset's S3 URI when the job completes, and exits with status 1 when it fails
* `--poll-secs <POLL_SECS>` — Seconds between status polls when `--wait` is set

  Default value: `30`
* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim depth status`

Check the status of a previously-issued depth job

**Usage:** `sim depth status [OPTIONS] --api-key <API_KEY> <JOB_ID>`

```text
Examples:
  sim depth status <job-id>
  sim depth status <job-id> --json
```

**Arguments:**

* `<JOB_ID>` — The job ID `sim depth measure` returned

**Options:**

* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim depth coverage`

Show which markets, venues and rungs a depth job would sample

**Usage:** `sim depth coverage [OPTIONS] --api-key <API_KEY>`

```text
Examples:
  # List the market ids `sim depth measure --market` accepts, per source.
  sim depth coverage
  sim depth coverage --json
```

**Options:**

* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--json` — Print output as JSON
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



## `sim datasets`

Get read access to the prebuilt and custom datasets

**Usage:** `sim datasets <COMMAND>`

**Subcommands:**

* `credentials` — Print temporary, read-only S3 credentials for the prebuilt datasets and your custom datasets



## `sim datasets credentials`

Print temporary, read-only S3 credentials for the prebuilt datasets and your custom datasets

**Usage:** `sim datasets credentials [OPTIONS] --api-key <API_KEY>`

```text
Examples:
  # Load the credentials into this shell, then list and download under the
  # `# datasets:` or `# your datasets:` URI it prints.
  eval "$(sim datasets credentials)"   # fish: sim datasets credentials | source
  aws s3 ls s3://<bucket>/datasets/
  aws s3 sync s3://<bucket>/datasets/<dataset>/ ./<dataset>/

  # Or keep a profile that renews itself when the hour-long credentials expire.
  # SIMULATOR_API_KEY must be set wherever the profile is used.
  # ~/.aws/config:
  #   [profile termina-datasets]
  #   credential_process = sim datasets credentials --format process
  #   region = <region>
  aws s3 sync s3://<bucket>/datasets/<dataset>/ ./<dataset>/ \
    --profile termina-datasets
```

**Options:**

* `--format <FORMAT>` — How to print the credentials

  Default value: `env`

  Possible values:
  - `env`:
    Shell exports: `eval "$(sim datasets credentials)"` in bash/zsh, `sim datasets credentials | source` in fish. The dataset URIs and expiry go to stderr
  - `process`:
    An AWS `credential_process` document. Point a `~/.aws/config` profile at `sim datasets credentials --format process` (with the profile's own `region`) and the SDKs re-run it whenever the credentials expire
  - `json`:
    The response as the server returned it

* `--api-key <API_KEY>` — Your simulator API key. Set SIMULATOR_API_KEY to avoid passing it on every command

   Environment variable: `SIMULATOR_API_KEY`
* `--url <URL>` — Server host, or a full ws:// or wss:// URL. A bare host connects over TLS

   Environment variable: `SIMULATOR_URL`

  Default value: `simulator.termina.technology`



<hr/>

<small><i>
    This document was generated automatically by
    <a href="https://crates.io/crates/clap-markdown"><code>clap-markdown</code></a>.
</i></small>
