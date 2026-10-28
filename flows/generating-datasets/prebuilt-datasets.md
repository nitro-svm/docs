# Prebuilt Datasets

Like prebuilt bundles, a few prebuilt datasets are ready to download. Use them to analyze execution quality and liquidity across venues without building a replay pipeline.

They answer questions like: how deep each pool was at a given size, where a venue's quotes diverged from what the router saw, when two venues' quotes crossed, and which swaps a venue lost and at what price.

| Dataset | What it holds | Path under `s3://data.termina.technology/datasets/` |
| --- | --- | --- |
| Onchain Depths | Liquidity depth for each venue, measured by simulating fills against its onchain state. | `depth/onchain/slot/v<revision>/` and `depth/onchain/oracle/v<revision>/` |
| Onchain Crosses | Intervals in which buying on one venue and selling on another was profitable: an atomic arbitrage. Derived from the `oracle` depths. | `depth/onchain/crosses/v<revision>/` |
| Aggregator Depths | Liquidity depth for the same venues as quoted by the router (currently Jupiter Metis). | `depth/router/slot/v<revision>/` |
| Executed + Hypothetical Fills | Swaps that executed, alongside what each venue would have filled had it won the swap. | `estimated-fills/v2/` |

`<revision>` is the revision `sim depth coverage` prints for the dataset's source. Data published under an older revision stays at its own path.

The prebuilt datasets cover the markets `sim depth coverage` lists, over the dates published so far. Dates are added as they're measured, not on a schedule. List the dates a dataset holds once you have [credentials](#access):

```bash
aws s3 ls s3://data.termina.technology/datasets/depth/onchain/slot/v2/
```

For other dates, request a [custom dataset](custom-datasets.md).

## Access

The datasets are Parquet files on S3. `sim datasets credentials` gives you temporary, read-only credentials for them. You don't need an AWS account.

{% stepper %}
{% step %}
### Load Credentials

```bash
eval "$(sim datasets credentials)"    # fish: sim datasets credentials | source
```

This exports `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, `AWS_REGION` and `AWS_DEFAULT_REGION` into your shell, and prints where the datasets are:

```
# datasets: s3://data.termina.technology/datasets/
# your datasets: s3://data.termina.technology/userdata/<your_key_name>/datasets/
# expires: 2026-10-27T13:00:00Z
```

* `# datasets:` holds the prebuilt datasets.
* `# your datasets:` holds your [custom datasets](custom-datasets.md).
* The credentials expire after an hour. Run the command again for new ones, or use a profile (below).
{% endstep %}

{% step %}
### Download

List a dataset, then sync the dates and markets you need. For example, one week of SOL-USDC onchain depths:

```bash
aws s3 ls s3://data.termina.technology/datasets/depth/onchain/slot/

aws s3 sync s3://data.termina.technology/datasets/depth/onchain/slot/v2/ ./onchain-depths/ \
  --exclude "*" \
  --include "date=2026-10-0[1-7]/market=SOL-USDC/*"
```
{% endstep %}

{% step %}
### Load in pandas

The `date`, `market` and `venue` path segments become columns.

```python
import pandas as pd

df = pd.read_parquet("onchain-depths/", filters=[("venue", "=", "HumidiFi")])

# Base units to tokens: SOL has 9 decimals, USDC 6. Each rung is one size, so summarize per rung.
sol_to_usdc = df[df.input_token == "SOL"]
print((sol_to_usdc.out_amount / 1e6).groupby(sol_to_usdc.rung_index).median())
```

To read straight from S3 instead, install `s3fs` and pass the `s3://` path to `pd.read_parquet`. It uses the credentials you exported.
{% endstep %}
{% endstepper %}

### Use a Profile

A profile renews the credentials whenever they expire, so long jobs keep working. Add it to `~/.aws/config`:

```ini
[profile termina-datasets]
credential_process = sim datasets credentials --format process
region = <region>
```

* Get `<region>` from `sim datasets credentials --format json`.
* `SIMULATOR_API_KEY` must be set wherever the profile is used.
* Pass `--profile termina-datasets` to the AWS CLI, or `AWS_PROFILE=termina-datasets` to any AWS SDK.

### Without the CLI

Request the credentials from `POST https://simulator.termina.technology/datasets/credentials` with your API key in the `X-API-Key` header. The response is described in [HTTP Endpoints](../../overview/installation/api-reference/http-endpoints.md#post-datasets-credentials). From Rust, call `BacktestClient::dataset_credentials`, as the [`datasets`](https://github.com/nitro-svm/examples/tree/main/src/bin/datasets) example does.

<details>

<summary>example_access.py</summary>

```python
from urllib.parse import urlparse

import boto3
import requests

API_KEY = "<YOUR_API_KEY>"
AUTH_URL = "https://simulator.termina.technology/datasets/credentials"

resp = requests.post(AUTH_URL, headers={"X-API-Key": API_KEY}, timeout=30)
resp.raise_for_status()
creds = resp.json()

s3 = boto3.client(
    "s3",
    aws_access_key_id=creds["access_key_id"],
    aws_secret_access_key=creds["secret_access_key"],
    aws_session_token=creds["session_token"],
    # `region` can be null: boto3 then uses the region from your AWS config.
    region_name=creds.get("region"),
)

# s3_uri is the prebuilt datasets' root, e.g. s3://data.termina.technology/datasets/
root = urlparse(creds["s3_uri"])
bucket, prefix = root.netloc, root.path.lstrip("/")

# A listing returns at most 1,000 keys per page, so paginate.
pages = s3.get_paginator("list_objects_v2").paginate(
    Bucket=bucket, Prefix=f"{prefix}depth/onchain/slot/"
)
keys = [
    obj["Key"]
    for page in pages
    for obj in page.get("Contents", [])
    if obj["Key"].endswith(".parquet")
]
print(f"{len(keys)} files, first: {keys[:3]}")

s3.download_file(bucket, keys[0], "part.parquet")
```

</details>

## Layout

Every file follows one pattern:

```
<dataset path>/date=<YYYY-MM-DD>/market=<MARKET>/[venue=<VENUE>/]part-<first_slot>-<last_slot>.parquet
```

* `date` is the UTC date of the row's slot.
* `market` and `venue` are spelled as `sim depth coverage` lists them, e.g. `market=SOL-USDC/venue=HumidiFi`.
* The depth datasets have a `venue=` level. Onchain Crosses and Executed + Hypothetical Fills don't.
* `part-<first_slot>-<last_slot>` names the slot range the file covers, inclusive.
* Skip anything under `_empty/`: those are markers for ranges with no rows, not Parquet.

Token amounts are in base units. Rows name tokens by ticker; divide an amount by `10^decimals` of its token:

| Ticker | Mint | Decimals |
| --- | --- | --- |
| `SOL` | `So11111111111111111111111111111111111111112` | 9 |
| `USDC` | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` | 6 |
| `USDT` | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` | 6 |
| `BTC` (cbBTC) | `cbbtcf3aa214zXHbiAZQwf4122FBYbraNdFqgw4iMij` | 8 |
| `ETH` | `7vfCXTUXx5WJV5JADk17DUJ4ksgau7utNKj4b963voxs` | 8 |
| `HYPE` | `98sMhvDwXj1RQi5c5Mndm3vPe9cBqPrbLaufMXFNMh5g` | 9 |
| `NVDAx` | `Xsc9qvGR1efVDFGLrVsmkzv3qi45LTBjeUKSPmx9qEh` | 8 |
| `Mu` | `MUxEsUKSMACyw5fZf68wxf5FLnZVhtU9CwH8uNNGay1` | 6 |
| `SNDK` | `SNDKbwMUQvZhnLnxLduradgLHG5KrPuKwpnrkkGRhfH` | 6 |
| `SPCX` | `SPCXxcqXj6e5dJDVNovHN8744zkbhM2bYudU45BimGb` | 6 |
| `NBIS` | `NBiSF3UaVUFtRzHwAfxyHsBCAZWGEKnMpewAE4oh7BG` | 6 |

## Schemas

<details>

<summary>Onchain Depths</summary>

`depth/onchain/slot/v<revision>/date=<YYYY-MM-DD>/market=<MARKET>/venue=<VENUE>/part-<first_slot>-<last_slot>.parquet`

Each row is one size (a rung) traded through one venue, in one direction, at one moment: one `(slot, trigger, tx_index, input_token, rung_index)` within its file. The `slot` files sample every venue at the start of every slot, before the slot's transactions (`grid`). The `oracle` files sample at each oracle update that changes the venue's price.

Each direction has 18 rungs, sized in its input token. Rung 0 is the size below, and each rung doubles the one before, so the largest is 131,072 times rung 0:

| Input token | Rung 0 |
| --- | --- |
| `SOL`, `HYPE`, `USDC`, `NVDAx`, `Mu`, `SNDK`, `SPCX`, `NBIS` | 1 token |
| `ETH` | 0.01 ETH |
| `BTC` | 0.001 BTC |

| Column | Type | Note |
| --- | --- | --- |
| `slot` | uint64 | |
| `trigger` | string | What fired the sample. `slot` files: `grid` (every slot; earlier revisions also have `event`, after a batch that ran the venue's program). `oracle` files: `snapshot` (the first slot of each date), `update` (after an oracle update) or `event` (after a batch where the venue's program wrote an account the probe reads; an `update` in the same batch takes its place). |
| `tx_index` | uint32, nullable | Index in the block of the transaction that fired the sample. Null for `grid` and `snapshot`. |
| `input_token` | string | Ticker, e.g. `SOL`. |
| `output_token` | string | Ticker. |
| `rung_index` | uint16 | The size's position on the ladder. Rung 0 is the smallest, and each rung doubles the one before. |
| `in_amount` | uint64 | The size, in base units of `input_token`. |
| `out_amount` | uint64, nullable | What the venue paid out. Null when the venue reverted at this size. |
| `price_impact_bps` | float64, nullable | How much worse this rung's rate is than rung 0's in the same sample. Positive is worse. Null when this rung or rung 0 didn't fill. |
| `update_signature` | string, nullable | `oracle` files only. The signature of the oracle update that fired the sample. |

</details>

<details>

<summary>Onchain Crosses</summary>

`depth/onchain/crosses/v<revision>/date=<YYYY-MM-DD>/market=<MARKET>/part-<first_slot>-<last_slot>.parquet`

Each row is one interval in which `sell_venue`'s bid was above `buy_venue`'s ask at one size. Base is the market's first token (SOL in SOL-USDC), quote the second.

| Column | Type | Note |
| --- | --- | --- |
| `data_start`, `data_end` | uint64 | First and last slot with depth data on this date and market. The same on every row. |
| `buy_venue`, `sell_venue` | string | The venue to buy base on and the venue to sell it on. |
| `open_slot` | uint64 | Where the interval opened. |
| `open_tx_index` | uint32, nullable | Index in the block of the transaction that opened it. Null when it opened at a snapshot. |
| `open_update_signature` | string, nullable | The oracle update that opened it. |
| `opened_by` | string, nullable | Which leg re-priced to open it: `buy`, `sell` or `both`. Null at a snapshot. |
| `close_kind` | string | What closed it: `update` (an oracle update re-priced a leg), `snapshot`, `expiry` (a leg's quote expired), `hole` (depth data stopped covering a leg) or `end` (still open where the data ends). |
| `close_slot`, `close_tx_index`, `close_update_signature` | nullable | Where it closed, as for the open. Null when `close_kind` is `end`. |
| `closed_by` | string, nullable | Which leg closed it: `buy`, `sell` or `both`. |
| `duration_txs` | uint64, nullable | Transactions between the open and the close. |
| `open_edge_bps`, `max_edge_bps` | float64 | How far the bid was above the ask, at the open and at its widest. |
| `open_profit_amount`, `max_profit_amount` | uint64 | Profit from buying and selling at this size, in quote base units, at the open and at its largest. |
| `consensus_price` | float64, nullable | The median mid price, at this size, of the other venues quoting both sides at the open. |
| `consensus_venues` | uint8 | How many venues `consensus_price` is the median of. |
| `buy_gap_amount`, `sell_gap_amount` | int64, nullable | How far each leg priced from `consensus_price`, in quote base units. Positive favors the arbitrageur. They add up to `open_profit_amount`. |
| `buy_quoted_slot`, `sell_quoted_slot` | uint64 | Where each leg's quote was last re-priced, at or before the open. |
| `buy_quote_age_txs`, `sell_quote_age_txs` | uint64 | Transactions between that re-pricing and the open. |
| `buy_in_amount`, `buy_out_amount` | uint64 | The buy leg: quote in, base out. `buy_in_amount` is the interval's size. |
| `sell_in_amount`, `sell_out_amount` | uint64 | The sell leg: base in, quote out. |

</details>

<details>

<summary>Aggregator Depths</summary>

`depth/router/slot/v<revision>/date=<YYYY-MM-DD>/market=<MARKET>/venue=<VENUE>/part-<first_slot>-<last_slot>.parquet`

Each row is one pool, one USD size and one direction, as the router quoted it once per slot. The USD sizes are 100, 200, 500, 1,000, 2,000, 5,000, 10,000, 20,000, 50,000, 100,000, 200,000, 500,000 and 1,000,000.

| Column | Type | Note |
| --- | --- | --- |
| `slot` | uint64 | The slot whose starting state the quote prices. Joins the Onchain Depths rows of the same slot. |
| `router_slot` | uint64, nullable | The router's ingest slot when it answered. A gap from `slot` means the router was pricing older state. |
| `pool` | string | The pool's address. |
| `usd_value` | uint32 | The size asked for, in whole US dollars. |
| `input_token` | string | Ticker, e.g. `SOL`. |
| `output_token` | string | Ticker. |
| `in_amount` | uint64, nullable | What the router would take in, in base units. Null when it declined. |
| `out_amount` | uint64, nullable | What the router would pay out, in base units. Null when it declined. |
| `quote_error` | string, nullable | Why this direction has no price: the venue program's error, or that the router returned no quote for it. |
| `error` | string, nullable | Why the request produced no answer at all. Null when the router answered. |

</details>

<details>

<summary>Executed + Hypothetical Fills</summary>

`estimated-fills/v2/date=<YYYY-MM-DD>/market=<MARKET>/part-<first_slot>-<last_slot>.parquet`

Each row is one original swap hop priced through one venue. `venue` is a column, not a path segment.

| Column | Type | Note |
| --- | --- | --- |
| `slot` | uint64 | The slot the original swap executed in. |
| `block_time` | int64 | The block's Unix time, in seconds. |
| `tx_index` | uint32 | Index of the transaction in the block. |
| `ix_index` | uint32, nullable | Index of the swap's instruction in the transaction. |
| `leg_index` | uint32, nullable | Index of the hop within the swap's route. |
| `original_signature` | string, nullable | Signature of the original swap. |
| `quote_slot` | uint64 | The slot the venue was priced in, `quote_lag_blocks` blocks before `slot`. |
| `quote_tx_index` | uint32 | Where in `quote_slot` it was priced. |
| `quote_lag_blocks` | uint32 | Blocks between `quote_slot` and `slot`. |
| `original_failed` | bool | Whether the original swap failed onchain. |
| `venue` | string | The venue priced. |
| `template` | string | The venue template used, e.g. `humidifi/sol-to-usdc/via-jupiter`. |
| `swap_mode` | string | `ExactIn`: the input is fixed and the output varies. `ExactOut`: the output is fixed and the input varies. |
| `amount` | uint64 | The fixed side: the hop's input for `ExactIn`, its output for `ExactOut`. |
| `estimate_fill` | uint64, nullable | What the venue would have filled: its output for `ExactIn`, its input for `ExactOut`. Null unless `outcome` is `filled`. |
| `l1_fill` | uint64, nullable | What the original swap filled, on the same side as `estimate_fill`. Null when it filled nothing. |
| `outcome` | string | `filled`, `reverted`, `unattributed` (ran, but the output could not be read) or `unsimulatable` (could not run). |
| `category` | string, nullable | Why it failed. Null when `outcome` is `filled`. |
| `l1_program` | string, nullable | Program address of the venue that filled the original swap. |
| `input_mint`, `output_mint` | string | Mint addresses, base58. |
| `input_amount`, `output_amount` | uint64 | What the original swap's hop took in and paid out onchain. |
| `replay_input_amount`, `replay_output_amount` | uint64, nullable | The same hop when the original swap was replayed. |

</details>
