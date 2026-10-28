# Rerouting Directly to a Venue

To compare every historical fill against what a venue would've filled if it had won the swap, use the [`sim run` command](../replaying-the-market.md#run-a-baseline) with the reroutes set to `venue` and provide a template file. The template defines how the simulator should integrate with the venue and which token pair to include.

```bash
sim run \
  --start-slot <START_SLOT> \
  --end-slot <END_SLOT> \
  --reroute-order-flow venue \
  --venue-templates <FILE> \
  --reroute-aggregators jupiter,okx,titan,dflow
```

Only the swaps of the aggregators in `--reroute-aggregators` are priced. It defaults to Jupiter, so name every aggregator whose swaps you want priced.

Each probe is simulated at the original swap's [pricing point](README.md#interpreting-the-numbers) and then discarded: the original transactions still run as they did onchain, so the replay never changes. To run both modes in one session, use `--reroute-order-flow router,venue`.

This shows whether the venue would've won on price but not whether a router would've actually routed the swap to it.

It's typically used when the venue isn't integrated with any aggregator yet, but it's also useful for comparing the competitiveness of a venue's prices independent of aggregator routing decisions. In other words, it can be thought of as _"true reality"_ instead of _"router's view of reality"_.

## Templates

A template is one swap instruction on your venue, for one direction of one token pair. For every hop of an original swap with the same `mode`, `inputMint` and `outputMint`, `sim` writes the hop's amount into the instruction, sets the taker's accounts and simulates it.

<details>

<summary>template.json</summary>

```json
[
  {
    "label": "my-venue/my-pool/usdc-to-token",
    "mode": "ExactIn",
    "inputMint": "<USDC_MINT>",
    "outputMint": "<TOKEN_MINT>",
    "instruction": {
      "programId": "<PROGRAM_ID>",
      "accounts": [
        {"address": "11111111111111111111111111111111", "writable": true, "signer": true},
        {"address": "11111111111111111111111111111111", "writable": true},
        {"address": "11111111111111111111111111111111", "writable": true},
        {"address": "<POOL_ADDRESS>", "writable": true},
        {"address": "<VAULT_A>", "writable": true},
        {"address": "<VAULT_B>", "writable": true},
        {"address": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA"}
      ],
      "data": {"data": "<BASE64_INSTRUCTION_DATA>", "encoding": "base64"}
    },
    "amountOffset": 8,
    "taker": {"user": [0], "inputTokenAccount": 1, "outputTokenAccount": 2}
  }
]
```

</details>

### Build a Template

1. Find an onchain transaction that swaps on your pool in the direction and mode you want to price.
2. Copy its swap instruction: `programId`, the `accounts` in order with their `writable` and `signer` flags, and the instruction data as base64. `getTransaction` with `"encoding": "jsonParsed"` and `"maxSupportedTransactionVersion": 0` lists every account, lookup-table accounts included, with its flags; it gives the data as base58, so convert it. Accounts that depend on the pool's price, such as a concentrated-liquidity pool's tick arrays, stay as copied, so probes revert once the price moves past them: copy a transaction that traded near the prices you want to probe.
3. Find the swap amount in the data, a little-endian u64. Its byte offset is `amountOffset`. Loosen any slippage limit in the data, such as a minimum output, so probes don't revert on a limit copied from that transaction.
4. Set the taker's accounts to `11111111111111111111111111111111` and list their indices in `taker`: the signer in `user`, its token accounts in `inputTokenAccount` and `outputTokenAccount`. `sim` replaces them with the original swap's fee payer and its token accounts.
5. If your program derives accounts from the taker, list them in `taker.derived`. See [File Formats](../../cli/file-formats.md#fields).
6. Add a second template, with its own `label`, for the other direction.
7. Test the file on a short range. If the `Venue probe` line shows only `reverted_<reason>` or `unsimulatable`, the template is wrong.

Every field and validation rule is in [File Formats](../../cli/file-formats.md#venue-templates).

The [templates that ship with `simulator-client`](https://docs.rs/crate/simulator-client/latest/source/recipes/venues.json) cover the pools the client supports, mostly in both directions, and work as starting points. Their labels follow `<venue>/<pool>/<input>-to-<output>`, sometimes with a suffix: a `via-jupiter` template calls the venue through Jupiter's route instruction. From Rust, pass [`simulator_client::recipes::VENUES`](https://docs.rs/simulator-client/latest/simulator_client/recipes/constant.VENUES.html)`.templates()` to `CreateSession`'s `reroute_direct_fill`, set `reroute_order_flow(true)`, and set `reroute_requote(false)` to skip rerouting via an aggregator. The [`reroute_venue_templates`](https://github.com/nitro-svm/examples/tree/main/src/bin/reroute_venue_templates) example runs the bundled templates or your own file and scores each one, and [`counterfactual_capital`](https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_capital) prices hops through templates while changing what a venue holds.

## Reading the Results

When the run ends, `sim` prints one `Venue probe` line per template, then the total, and where it saved the results:

```
↻ Reroute: 1204 detected → 0 rerouted → 0 simulated → 0 succeeded | 98 filtered | 0 reroute-fail
◆ Venue probe: my-venue/my-pool/usdc-to-token: 412 matched → filled 398, reverted_<reason> 9, unattributed 5 | 398 scored, +3.2 bps
◆ Venue probe: total: 412 matched → filled 398, reverted_<reason> 9, unattributed 5 | 398 scored, +3.2 bps
↻ Reroute tables: s3://data.termina.technology/userdata/<your_key_name>/datasets/reroute/<run_name>.sqlite (published once the session shuts down, kept 7 days)
```

When rerouting directly to a venue, only `detected` and `filtered` move on the `Reroute` line. `filtered` counts swaps that share a transaction with an aggregator not in `--reroute-aggregators`; they aren't probed.

On a `Venue probe` line:

* `matched`: hops of original swaps the template matched. Each one is probed once per matching template.
* The outcomes: `filled`, `reverted_<reason>`, `unattributed` (ran, but the measured token account was gone or unreadable afterwards) and `unsimulatable` (couldn't be built or run).
* `scored` and the mean bps: the template's fill against the original hop's onchain fill, with every hop counting once. Positive means your venue beat the original swap. `ExactIn` compares the output received, `ExactOut` the input charged.

{% hint style="info" %}
Without `--program-id`, the run records no transactions, so it ends with `Simulation finished with no transactions` and exit status 1. The reroute results are still printed and saved.
{% endhint %}

### Query the SQLite File

Load your dataset credentials and download the file the run printed:

```bash
eval "$(sim datasets credentials)"   # fish: sim datasets credentials | source
aws s3 cp s3://data.termina.technology/userdata/<your_key_name>/datasets/reroute/<run_name>.sqlite .
sqlite3 <run_name>.sqlite
```

`direct_fill_probes` has one row per probe: `template` (the `label`), `outcome`, `category` (the revert reason), `template_fill`, `l1_fill` and `change_vs_l1_bps`. The hop's mints and amounts are on its `original_legs` row.

Win rate and mean basis points per template, with every hop counting once:

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

This mean can disagree in sign with the mean weighted by size, and it includes how the state moved between the pricing point and execution. [Interpreting the Numbers](README.md#interpreting-the-numbers) has the weighted mean and the comparison on the same state.

Why probes reverted:

```sql
SELECT template, category, COUNT(*) AS probes
FROM direct_fill_probes
WHERE outcome = 'reverted'
GROUP BY template, category
ORDER BY probes DESC;
```

Every table and column is described in [Output Format](../../cli/output-format.md#reroute-results).
