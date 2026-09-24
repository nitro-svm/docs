# Custom Datasets

{% stepper %}
{% step %}
### Verify Coverage

Custom datasets can be generated for any slot range, market, or venue that the [prebuilt datasets](prebuilt-datasets.md) don't cover.&#x20;

They rely on [bundles](../replaying-the-market.md#select-a-time-window), which contain the historical transactions for a slot range. If the desired range isn't covered by a prebuilt bundle, build a custom bundle before generating the dataset.&#x20;
{% endstep %}

{% step %}
### Capture Depth Data

By default, depth is measured by simulating fills against each venue's onchain state. To capture Jupiter Metis' view of the same venues instead, set `--source router`. Use the `--market` flag to select specific token pairs.

Program binaries can also be swapped during capture, the same as in [market replay](../replaying-the-market.md#run-an-experiment), to measure how new logic changes the results.

```bash
sim depth measure \
--source onchain \
--start-slot 447458502 \
--end-slot 447458702 \
--market SOL-USDC \
--market BTC-USDC
```

```
issued: 9c30e2a3-9145-4d35-917d-03a7c709d076 (onchain, 447458502-447458702) in_progress
```
{% endstep %}

{% step %}
### Monitor Status

Since the data generation runs on the cloud, use the job ID from the previous output to check  progress.

```bash
sim depth status 9c30e2a3-9145-4d35-917d-03a7c709d076
```

```
status: 9c30e2a3-9145-4d35-917d-03a7c709d076 (onchain, 447458502-447458702) completed
```
{% endstep %}

{% step %}
### Access the Dataset

Once the job completes, the dataset is available on S3 using the same [credentials flow](prebuilt-datasets.md#access) as the prebuilt datasets. Custom datasets are scoped to the requesting API key and aren't available to other users.
{% endstep %}

{% step %}
### Beyond Depth

Datasets aren't limited to depth. The [Rust client](../../overview/installation/rust-reference.md) can probe state and subscribe to account updates during relay, which makes it possible to capture any metric, such as oracle prices, vault balances, or fee accrual.&#x20;
{% endstep %}
{% endstepper %}
