# Custom Datasets

Request a custom dataset for a slot range the [prebuilt datasets](prebuilt-datasets.md) don't cover, over the markets `sim depth coverage` lists. You can request Onchain Depths (`--source onchain`) and Aggregator Depths (`--source router`); the other prebuilt datasets can't be requested. A custom dataset has the same layout and schema as the prebuilt one, and only your API key can read it.

{% hint style="info" %}
Custom datasets need to be enabled on your API key: [contact us](../../README.md#get-connected). A key without them gets `HTTP 403` with `generate_depth feature not enabled`. Depth jobs and custom bundle builds share one request limit per key; over it, a request gets `HTTP 429`.
{% endhint %}

The steps below use `sim`. The [`datasets`](https://github.com/nitro-svm/examples/tree/main/src/bin/datasets) example does the same from Rust.

{% stepper %}
{% step %}
### Check Coverage

List the markets and venues each source samples:

```bash
sim depth coverage
```

```
onchain (revision 2): 9 market(s) x 21 venue(s) at 18 rung(s)
  markets: SOL-USDC, BTC-USDC, ETH-USDC, HYPE-USDC, NVDAx-USDC, Mu-USDC, SNDK-USDC, SPCX-USDC, NBIS-USDC
  venues : BisonFi, Tessera, Quantum, GoonFi, HumidiFi, Scorch, ZeroFi, SolFi, Flux, ...
  published: none yet
router (revision 2): ...
```

* `markets` are the IDs `--market` takes.
* `revision` is the `v<revision>` in the dataset's path.
* `published` lists the ranges your custom datasets already cover at this revision.
{% endstep %}

{% step %}
### Pick a Slot Range

Depth jobs take slots, not dates. Slot times changed during 2026, so look each date up rather than extrapolating from one slot:

* For a date the prebuilt datasets cover, its file names hold its slots: `aws s3 ls s3://data.termina.technology/datasets/depth/onchain/slot/v2/date=2026-09-01/market=SOL-USDC/venue=HumidiFi/` lists `part-<first_slot>-<last_slot>.parquet` files.
* For any date, ask a Solana RPC node: the script below finds the first slot at or after a UTC time with `getBlockTime`. It uses the public mainnet RPC, which rate-limits, so a lookup takes about a minute; point `RPC` at your own node to go faster.

<details>

<summary>slot_at.py</summary>

```python
import json
import sys
import time
import urllib.error
import urllib.request
from datetime import datetime, timezone

RPC = "https://api.mainnet-beta.solana.com"


def rpc(method, *params):
    body = json.dumps({"jsonrpc": "2.0", "id": 1, "method": method, "params": params})
    request = urllib.request.Request(RPC, body.encode(), {"Content-Type": "application/json"})
    while True:
        try:
            with urllib.request.urlopen(request) as response:
                return json.load(response).get("result")
        except urllib.error.HTTPError as error:
            if error.code != 429:
                raise
            time.sleep(5)  # rate-limited: wait, then retry


def block_time(slot):
    # A skipped slot has no block: take the next slot that has one.
    while (seconds := rpc("getBlockTime", slot)) is None:
        slot += 1
    return seconds


def slot_at(date):
    """The first slot produced at or after `date`, read as UTC."""
    target = datetime.fromisoformat(date).replace(tzinfo=timezone.utc).timestamp()
    low, high = 0, rpc("getSlot")
    while high - low > 1:
        middle = (low + high) // 2
        if block_time(middle) < target:
            low = middle
        else:
            high = middle
    return high


print(slot_at(sys.argv[1]))
```

</details>

```bash
python3 slot_at.py 2026-09-01   # 443282098
python3 slot_at.py 2026-09-08   # 445196370
```

Use the first date's slot as `--start-slot`, and the slot before the second date's as `--end-slot`.

* One job covers at most 10,000,000 slots.
* The range doesn't need a prebuilt bundle: the job builds the bundles it needs first.
* The job widens the range to the nearest snapshots before and after it. Snapshots are about an hour apart, so it can capture up to an hour of extra slots on each side.
{% endstep %}

{% step %}
### Request the Dataset

`--source` is required:

* `onchain` measures depth by simulating fills against each venue's onchain state.
* `router` captures the router's quotes for the same venues (currently Jupiter Metis).

```bash
sim depth measure \
  --source onchain \
  --start-slot 447458502 \
  --end-slot 447458702 \
  --market SOL-USDC \
  --market BTC-USDC
```

```
issued: 9c30e2a3-9145-4d35-917d-03a7c709d076 (onchain, 447458502-447458702) in_progress capture <first_slot>-<last_slot>
```

* Leave out `--market` to sample every market.
* Add `--granularity oracle` to sample after each oracle update and each batch where the venue's program wrote an account a probe reads, instead of once per slot. `onchain` only.
* Add `--wait` to wait for the job to finish.
* Sending the same request again (same source, granularity, range, markets and overrides) returns the same job ID and its status, without running or charging it again. A failed job runs again, also without a new charge. A request that differs in any of these, such as an overlapping range, is a new job.
{% endstep %}

{% step %}
### Monitor Status

The job runs on the server. Check on it with the job ID from the previous output:

```bash
sim depth status 9c30e2a3-9145-4d35-917d-03a7c709d076
```

```
status: 9c30e2a3-9145-4d35-917d-03a7c709d076 (onchain, 447458502-447458702) completed capture <first_slot>-<last_slot>
```

* The status is `in_progress`, `completed`, `failed` or `needs_investigation`. With `needs_investigation`, the outcome is unknown: don't assume the dataset is complete, and contact us with the job ID.
* `capture` is the range the job actually measured.
* A range without bundles waits for them to build before it is measured.
* `sim depth measure --wait` prints the dataset's S3 URI when the job completes, and exits with status 1 when it fails or needs investigation.
{% endstep %}

{% step %}
### Access the Dataset

Load credentials with `sim datasets credentials`, as for the [prebuilt datasets](prebuilt-datasets.md#access). Your datasets are under the `# your datasets:` URI it prints. If it prints none, your key's name can't be used as a storage path: contact us.

```
s3://data.termina.technology/userdata/<your_key_name>/datasets/
```

Below that URI, the paths match the prebuilt ones:

| Request | Path |
| --- | --- |
| `--source onchain` | `depth/onchain/slot/v<revision>/` |
| `--source onchain --granularity oracle` | `depth/onchain/oracle/v<revision>/` |
| `--source router` | `depth/router/slot/v<revision>/` |

```bash
eval "$(sim datasets credentials)"
aws s3 sync s3://data.termina.technology/userdata/<your_key_name>/datasets/depth/onchain/slot/v2/ ./my-depths/
```

The files and columns are described in [Prebuilt Datasets](prebuilt-datasets.md#layout).
{% endstep %}
{% endstepper %}

## Cost and Retention

* Each new request counts once against your key's request limit, which custom bundle builds share. The limit is set per key; over it, a request gets `HTTP 429`.
* Bundles a job builds for its range don't count against the limit.
* Sending the same request again, and the re-run of a failed job, don't count either.
* Custom datasets don't expire. Rerouting results under the same `datasets/` URI are kept for 7 days.

## Measure Modified Programs

With the `onchain` source, a job can run your own program binaries and account overrides in place of the deployed ones, the same as in [market replay](../replaying-the-market.md#run-an-experiment). This measures how new logic changes depth.

`sim depth measure` doesn't take them. Set `program_overrides` and `account_overrides` on the `DepthRequest` you pass to the [Rust client](../../overview/installation/rust-reference.md)'s `BacktestClient::generate_depth`, or on the body of [`POST /generate-depth`](../../overview/installation/api-reference/http-endpoints.md#post-generate-depth).

The job's status then carries a `modification_digest`, and the dataset is published apart from the unmodified one:

```
s3://data.termina.technology/userdata/<your_key_name>/datasets/modified/depth/onchain/slot/<modification_digest>/v<revision>/
```

{% hint style="info" %}
### Beyond Depth

Datasets aren't limited to depth. The [Rust client](../../overview/installation/rust-reference.md) can probe state and subscribe to account updates during replay, which makes it possible to capture any metric, such as oracle prices, vault balances, or fee accrual.
{% endhint %}
