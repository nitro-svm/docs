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
2026-06-30T17:53:10Z           2026-06-30T19:00:53Z           429,926,541     429,936,540     10,000          account_state   global
2026-06-30T18:23:38Z           2026-06-30T18:27:02Z           429,931,041     429,931,540     500             account_state   global
2026-07-19T05:50:16Z           2026-07-19T07:00:13Z           433,838,452     433,848,451     10,000          account_state   global
```

For ranges older than 30 days or ones that fall between prebuilt bundles, build a custom one instead.

```bash
sim bundle build --start-slot 400000000 --end-slot 400001000
```
{% endstep %}

{% step %}
### Run a Baseline

Start a session to replay transactions and capture the existing behavior. If the range is covered by multiple bundles, use `--parallel` to split the work between concurrent sessions.

```bash
# Baseline
sim run \
  --start-slot 400000000 \
  --end-slot 400001000 \
  --program-id <PROGRAM_ID> \
  --output-file baseline.json
```

This generates a newline-delimited JSON file that contains the results of the run.

<details>

<summary>baseline.json</summary>

```json
{
  "metadata": {
    "start_slot": 400000000,
    "end_slot": 400001000,
    "program_id": "<PROGRAM_ID>",
    "session_ids": ["..."],
    "timestamp": "2026-05-29T03:06:07Z"
  },
  "transactions": [
    {
      "slot": 400000042,
      "signature": "<BASE58_SIG>",
      "success": true,
      "error": null,
      "logs": ["Program log: ...", "..."],
      "sol_changes": { "<PUBKEY>": -5000000 },
      "token_changes": { "<PUBKEY>": { "<MINT>": -1000 } },
      "account_diffs": { "<PUBKEY>": { "before": "...", "after": "..." } }
    }
  ],
  "summary": {
    "total": 847,
    "successes": 831,
    "failures": 16
  }
}
```

</details>

`sim` provides a helper that reads the JSON to summarize the run's output.

```bash
sim summarize baseline.json
```

```
File         : baseline.json
Slot range   : 400000000 → 400001000
Program IDs  : <PROGRAM_ID>
Session(s)   : 10
Ran at       : 2026-05-29 03:06:07 UTC

i Transactions : 205308  (96972 successes, 108336 failures)
```

```bash
# Trace specific wallet balance changes
sim summarize baseline.json --accounts E2gE...
```

```
Account: E2gE...
  SOL 29.455253127 → 29.293640983  (Δ -0.161612144)
```
{% endstep %}

{% step %}
### Run an Experiment

Build a modified program with parameter changes, feature upgrades, or bug fixes. Substitute it in place of the original binary and run the new logic against the same historical flow.

```bash
# Experiment
sim run \
  --start-slot 400000000 \
  --end-slot 400001000 \
  --program-id <PROGRAM_ID> \
  --program-so <PATH_TO_NEW_BINARY> \
  --output-file experiment.json
```

If the new program requires higher compute units, bump it up to prevent transaction failures with `--extra-compute-units <EXTRA_UNITS>`.
{% endstep %}

{% step %}
### Compare Results

Once both the baseline and experiment are complete, compare the two runs:

* regressions: original transaction that succeeded -> now fails
* improvements: original transaction that failed -> now succeeds
* balance changes: deltas in every account's token accounts

```bash
sim compare baseline.json experiment.json
```

```bash
# Only show what broke
sim compare baseline.json experiment.json regressions
```

```bash
# Check P&L impact
sim compare baseline.json experiment.json balances
```
{% endstep %}
{% endstepper %}
