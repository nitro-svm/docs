# Rerouting Historical Flow

For onchain market makers, it's hard to predict how a change to quoting logic will affect taker flow. Rerouting historical swaps against the updated logic makes it easier to see how often it would've won and at what prices compared to competing pools.

There are two modes: via an aggregator or directly to a venue.

Both modes run with `sim run --reroute-order-flow`. A run prints a reroute summary when it ends and saves per-swap results as a SQLite file, kept for 7 days. Each mode's page explains how to read them; [Interpreting the Numbers](#interpreting-the-numbers) says what each one compares.

<table data-view="cards"><thead><tr><th></th><th data-hidden data-card-target data-type="content-ref"></th><th data-hidden data-card-cover data-type="image">Cover image</th></tr></thead><tbody><tr><td>Rerouting via an Aggregator</td><td><a href="rerouting-via-an-aggregator.md">rerouting-via-an-aggregator.md</a></td><td><a href="../../.gitbook/assets/5 (3).png">5 (3).png</a></td></tr><tr><td>Rerouting Directly to a Venue</td><td><a href="rerouting-directly-to-a-venue.md">rerouting-directly-to-a-venue.md</a></td><td><a href="../../.gitbook/assets/4 (2).png">4 (2).png</a></td></tr></tbody></table>



## Names You'll See

Each mode has one name in these docs, and its own identifiers in the CLI, the Rust client, the wire format and the results:

| Where | Rerouting via an aggregator | Rerouting directly to a venue |
| --- | --- | --- |
| `sim run --reroute-order-flow` | `router` (the default) | `venue` |
| Rust `CreateSession` | `reroute_requote` (on by default) | `reroute_direct_fill` |
| Wire `reroute` object | `requote` | `directFill` |
| Notifications | `Requote` | `DirectFill` |
| SQLite tables | `reroute_quotes`, `reroute_run_metrics` | `direct_fill_probes`, `direct_fill_run_metrics` |
| End-of-run summary | the `Reroute` line | the `Venue probe` lines |

Both modes also replay each original swap as a baseline: the `Original` notification, the `original_sims` table and the `replay` rows of `original_legs`. `l1` in a flag or column name means the original swap as it ran onchain.

## Interpreting the Numbers

Every original swap is priced at its **pricing point**: the replay's position `quote_lag_blocks` blocks before the block the swap executed in. That lag is the age of the original transaction's recent blockhash, so the reroute sees state about as old as the original taker did. `original_swaps` records it as `quote_slot`, `quote_batch` and `quote_lag_blocks`.

* **Via an aggregator:** the router quotes the swap at its pricing point. The new transaction is then simulated just before the original swap, in the original's block.
* **Directly to a venue:** each template probe is simulated at the pricing point.
* **Both modes:** the original swap is replayed at the pricing point.

Every simulation is discarded. The original transactions still run as they did onchain, so rerouting never changes the replay.

What each number compares:

| Number | Compares | Measures |
| --- | --- | --- |
| `direct_fill_probes.change_vs_l1_bps` | The probe against the hop's onchain fill | The venue's edge, plus how the state moved between the pricing point and execution |
| `original_sims.change_vs_l1_bps` | The replayed original against its onchain fill | How the state moved alone |
| Probe against the `replay` hop (query below) | Both at the pricing point | The venue's edge alone |
| `reroute_quotes.output_bps` | The router's quote against the original instruction's quoted output | Quote against quote |
| `realized_output_amount` against `original_realized_output_amount` | The new transaction against the original, onchain | Fill against fill |

Positive basis points mean more output than the number compared against, or, for `ExactOut`, less input charged. Every query below reads the SQLite file a run saves; see [Output Format](../../cli/output-format.md#reroute-results) for every column.

A probe's change against the onchain fill, minus how much the state moved for the same swap, joined on `(original_signature, ix_index)`:

```sql
SELECT p.template,
       COUNT(*)                                     AS pairs,
       AVG(o.change_vs_l1_bps)                      AS drift_bps,
       AVG(p.change_vs_l1_bps - o.change_vs_l1_bps) AS edge_bps
FROM direct_fill_probes p
JOIN original_sims o
  ON o.original_signature = p.original_signature AND o.ix_index = p.ix_index
WHERE p.outcome = 'filled' AND o.outcome = 'filled'
  AND NOT EXISTS (
    SELECT 1 FROM original_legs l
    WHERE l.original_signature = p.original_signature AND l.ix_index = p.ix_index
      AND l.source = 'l1' AND l.hop_index > 0)
GROUP BY p.template;
```

`edge_bps` is the venue's edge. The query keeps swaps with one hop only: `original_sims` covers the whole swap, so on a multi-hop swap it also moves with hops the probe never priced, and can flip the sign. For any swap, compare each probe with the same hop replayed on the same state. This is `ExactIn` only; the replayed hop is rescaled to the probe's size, since an earlier hop can fill differently in the replay:

```sql
WITH probe AS (
  SELECT p.template, p.template_fill,
         -- The replayed hop's output, rescaled to the probe's input.
         r.output_amount * 1.0 * l.input_amount / r.input_amount AS same_state_fill
  FROM direct_fill_probes p
  JOIN original_legs l
    ON l.original_signature = p.original_signature AND l.ix_index = p.ix_index
   AND l.leg_index = p.leg_index AND l.source = 'l1'
  JOIN original_legs r
    ON r.original_signature = l.original_signature AND r.ix_index = l.ix_index
   AND r.hop_index = l.hop_index AND r.share_index = l.share_index AND r.source = 'replay'
  WHERE p.outcome = 'filled' AND p.swap_mode = 'ExactIn'
    AND r.input_amount > 0 AND r.output_amount > 0
)
SELECT template,
       COUNT(*) AS pairs,
       AVG(10000.0 * (template_fill - same_state_fill) / same_state_fill) AS same_state_bps
FROM probe
GROUP BY template;
```

A probe names its hop by `(original_signature, ix_index, leg_index)`, and its `original_legs` row with `source = 'l1'` adds `hop_index` and `share_index`.

`AVG(change_vs_l1_bps)` counts every hop once. Weight by size instead, since many small wins can hide a few large losses and flip the sign. A template trades one pair in one direction, so weighting by the hop's input amount weights by size in one token:

```sql
SELECT p.template,
       COUNT(*) AS scored,
       SUM(p.change_vs_l1_bps * 1.0 * l.input_amount) / SUM(l.input_amount) AS weighted_bps
FROM direct_fill_probes p
JOIN original_legs l
  ON l.original_signature = p.original_signature AND l.ix_index = p.ix_index
 AND l.leg_index = p.leg_index AND l.source = 'l1'
WHERE p.outcome = 'filled' AND p.change_vs_l1_bps IS NOT NULL AND p.swap_mode = 'ExactIn'
GROUP BY p.template;
```

For `ExactOut` templates, weight by `l.output_amount` instead.
