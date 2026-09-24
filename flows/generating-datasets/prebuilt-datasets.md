# Prebuilt Datasets

### Data

Similar to bundles, a few prebuilt datasets are available off-the-shelf. They make it possible to analyze execution quality and liquidity across venues without building a replay pipeline, even for teams new to Solana's programming model.

More specifically, these datasets can answer questions like: where a venue's quotes diverge from what aggregators see, how deep each pool actually is at a given size, and which swaps a venue lost and at what price.

| Dataset                       | Description                                                                                   | Download Path                   |
| ----------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------- |
| Onchain Depths                | Liquidity depth for each venue, computed directly from onchain pool state.                    | `datasets/depth/v2/`            |
| Aggregator Depths             | Liquidity depth for the same venues as quoted by Jupiter Metis.                               | `datasets/depth-diagnostic/v2/` |
| Executed + Hypothetical Fills | Fills that were executed, alongside fills each venue would have received had it won the swap. | `datasets/estimated_fills/v2/`  |

### Schemas

All datasets are stored as Parquet, partitioned by date, market, and venue. The path pattern for each dataset is in each section.

Token amounts are shown in their base units (e.g. SOL is 9 decimals, BTC is 8, USDC is 6).

<details>

<summary>Onchain Depths</summary>

`datasets/depth/v{version_number}/date=YYYY-MM-DD/market={token_pair}/venue={venue}/part-{start_slot}-{end_slot}.parquet`

<table><thead><tr><th width="236.91796875">Column</th><th>Note</th></tr></thead><tbody><tr><td>slot</td><td></td></tr><tr><td>input_token</td><td>ticker (e.g. SOL)</td></tr><tr><td>output_token</td><td></td></tr><tr><td>in_amount</td><td></td></tr><tr><td>out_amount</td><td></td></tr><tr><td>price_impact_bps</td><td>rate compared against rung 0</td></tr><tr><td>trigger</td><td><ul><li>by grid: always once per slot</li><li>by event: can be intra-slot or once every few slots</li></ul></td></tr><tr><td>rung_index</td><td>rung sizes double each step; rung 0 is the anchor</td></tr></tbody></table>

</details>

<details>

<summary>Aggregator Depths</summary>

`datasets/depth-diagnostic/v{version_number}/date=YYYY-MM-DD/market={token_pair}/venue={venue}/part-{start_slot}-{end_slot}.parquet`

<table><thead><tr><th width="223.55859375">Column</th><th>Note</th></tr></thead><tbody><tr><td>slot</td><td></td></tr><tr><td>router_slot</td><td>Jupiter Metis' ingest slot when it responded (this may lag behind  <code>slot</code>)</td></tr><tr><td>pool</td><td>address for the pool</td></tr><tr><td>router_label</td><td>human-readable label for the pool (e.g. SolFi V2)</td></tr><tr><td>input_token</td><td>ticker (e.g. SOL)</td></tr><tr><td>output_token</td><td></td></tr><tr><td>in_amount</td><td></td></tr><tr><td>out_amount</td><td></td></tr><tr><td>usd_value</td><td><code>in_amount</code> in USD</td></tr><tr><td>account_hashes</td><td>fingerprint per backing account Metis priced from</td></tr><tr><td>quote_error</td><td>why Metis wasn't reachable</td></tr><tr><td>error</td><td>why Metis returned no price (e.g. pool didn't have enough liquidity)</td></tr><tr><td>raw_entry</td><td>Metis full json response</td></tr></tbody></table>

</details>

<details>

<summary>Executed + Hypothetical Fills</summary>

`datasets/estimated_fills/v1/date=YYYY-MM-DD/template_program={program_id}/part-{start_slot}-{end_slot}.parquet`

<table><thead><tr><th width="284.50390625">Column</th><th>Description</th></tr></thead><tbody><tr><td>slot</td><td></td></tr><tr><td>block_time</td><td>timestamp</td></tr><tr><td>swap_mode</td><td><ul><li>exact in: input amount is fixed and output varies (e.g. sell exactly 10 USDC)</li><li>exact out: output amount is fixed and input varies (e.g. receive exactly 1 SOL)</li></ul></td></tr><tr><td>input_mint</td><td>ticker (e.g. SOL)</td></tr><tr><td>output_mint</td><td></td></tr><tr><td>amount</td><td>for the original swap</td></tr><tr><td>l1_fill</td><td>for the original swap</td></tr><tr><td>l1_program</td><td>address of the DEX that filled the original swap</td></tr><tr><td>l1_venue</td><td>human-readable name of the DEX</td></tr><tr><td>input_amount</td><td><ul><li>matches <code>l1_fill</code> if <code>swap_mode</code> is exact out</li><li>matches <code>amount</code> if <code>swap_mode</code> is exact in</li></ul></td></tr><tr><td>output_amount</td><td><ul><li>matches <code>l1_fill</code> if <code>swap_mode</code> is </li><li>matches <code>amount</code> if <code>swap_mode</code> is  </li></ul></td></tr><tr><td>original_failed</td><td>whether the original swap succeeded or failed</td></tr><tr><td>quote_lag_blocks</td><td></td></tr><tr><td>estimate_fill</td><td>simulated venue's version of <code>l1_fill</code></td></tr><tr><td>replay_input_amount</td><td>simulated venue's <code>input_amount</code></td></tr><tr><td>replay_output_amount</td><td>simulated venue's <code>output_amount</code></td></tr><tr><td>outcome</td><td><ul><li>result of the simulated swap</li><li>either filled, reverted, unattributed, unsimulatable</li></ul></td></tr><tr><td>category</td><td>revert reason for simulated swap </td></tr></tbody></table>

</details>

### Access

Datasets are hosted on S3 as Parquet files. Request credentials from the authentication URL below, then access the bucket directly, as shown in the example code.

<table><thead><tr><th width="170.87890625"></th><th>URL</th></tr></thead><tbody><tr><td>Dataset Authentication</td><td><pre><code>https://simulator.termina.technology/datasets/credentials
</code></pre></td></tr><tr><td>S3 Bucket</td><td><pre><code>s3://data.termina.technology
</code></pre></td></tr></tbody></table>

<details>

<summary>example_access.py</summary>

```python
import boto3
import requests

API_KEY = ""
AUTH_URL = "https://simulator.termina.technology/datasets/credentials"
DATA_URL = "data.termina.technology"

# get access credentials
creds = requests.post(AUTH_URL, headers={"X-API-Key": API_KEY}).json()
print(f"creds: {creds.keys()}\n")

# set up an authenticated s3 client
s3 = boto3.client(
    "s3",
    aws_access_key_id=creds["access_key_id"],
    aws_secret_access_key=creds["secret_access_key"],
    aws_session_token=creds["session_token"],
    region_name=creds["region"],
)

# list every parquet file in the datasets
resp = s3.list_objects_v2(Bucket=DATA_URL, Prefix="datasets/")
files = [obj["Key"] for obj in resp.get("Contents", [])]
print(files)
```



</details>
