# CLI Client

`sim` is a command-line client for managing simulation sessions without writing code. The tool allows replaying historical slots, injecting modified programs, and analyzing transaction outcomes.

### Installation

Pre-built binaries are available for Linux and macOS (Apple Silicon):

```bash
curl -fsSL https://cli.simulator.termina.technology/install.sh | bash
```

Update to the latest version in place:

```bash
sim update
```

All commands that connect to the simulator require an API key. Pass it via the `--api-key` flag or the `SIMULATOR_API_KEY` environment variable:

```bash
export SIMULATOR_API_KEY=<API_KEY>
```

### Workflow

1. **Check available slots:** `sim ranges` to find a recent slot range to test against.
2. **Run a baseline:** `sim run` + `--program-id` to capture the current behavior.
3. **Build a modified program:** `cargo build-sbf` (or equivalent).
4. **Run with the override:** `sim run` + `--program-so` pointing at the new binary.
5. **Compare the runs:** `sim compare baseline.json experiment.json` to see regressions, improvements, and balance changes.

### Commands

#### `sim ranges`: List Available Slot Ranges

Query for which slot ranges are available to backtest.

```
sim ranges [OPTIONS]
```

```bash
# List all available ranges
sim ranges

# Filter to a specific date window
sim ranges --after 2026-01-01 --before 2026-03-01
```

#### `sim run`: Run a Simulation

Start a simulation session, replay transactions, and stream results to an output file.

```
sim run [OPTIONS] --start-slot <SLOT> --end-slot <SLOT>
```

Run a baseline, then re-run with a modified program binary to compare outcomes:

```bash
# Baseline
sim run \
  --start-slot 300000000 \
  --end-slot 300001000 \
  --program-id <PROGRAM_ID> \
  --output-file baseline.json

# With modified program
sim run \
  --start-slot 300000000 \
  --end-slot 300001000 \
  --program-id <PROGRAM_ID> \
  --program-so ./target/deploy/program.so \
  --output-file experiment.json
```

For large slot ranges, `--parallel` splits the work across up to 10 concurrent sessions:

```bash
sim run \
  --start-slot 300000000 \
  --end-slot 300010000 \
  --parallel
```

#### `sim compare`: Diff Two Simulation Runs

Compare a baseline run against an experiment run and highlight differences.

```
sim compare <BASELINE> <EXPERIMENT> [SECTION]
```

```bash
sim compare baseline.json experiment.json

# Only show what broke:
sim compare regressions baseline.json experiment.json

# Check P&L impact:
sim compare balances baseline.json experiment.json
```

#### `sim summarize`: Summarize Simulation Outputs

Show metadata and aggregate stats for a simulation output file. Optionally trace balances for specific accounts across all transactions.

```
sim summarize <FILE> [--accounts <ADDRS>]
```

```bash
sim summarize baseline.json

# Trace specific wallets:
sim summarize baseline.json --accounts <WALLET1>,<WALLET2>
```

#### `sim update`: Update the CLI

Downloads and replaces the current `sim` binary with the latest published version.

```bash
sim update
```

### Output Format

`sim run` writes a JSON file with the following structure:

```json
{
  "metadata": {
    "start_slot": 300000000,
    "end_slot": 300001000,
    "program_id": "<PROGRAM_ID>",
    "session_ids": ["..."],
    "timestamp": "2025-01-15T12:00:00Z"
  },
  "transactions": [
    {
      "slot": 300000042,
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

This file is the input for `sim compare` and `sim summarize`.
