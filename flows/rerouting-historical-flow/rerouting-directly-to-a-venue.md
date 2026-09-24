# Rerouting Directly to a Venue

To compare every historical fill against what a venue would've filled if it had won the swap, use the [`sim run` command](../replaying-the-market.md#run-a-baseline) with the reroutes set to `venue` and provide a template file. The template defines how the simulator should integrate with the venue and which token pair to include.

```bash
sim run \
  --start-slot 400000000 \
  --end-slot 400010000 \
  --reroute-order-flow venue \
  --venue-templates <FILE>
```

<details>

<summary>template.json</summary>

```json
[
  {
    "label": "pool/my-pool/usdc-to-token",
    "mode": "ExactIn",
    "inputMint": "<USDC>",
    "outputMint": "<TOKEN>",
    "instruction": {
      "programId": "<PROGRAM_ID>",
      "accounts": [
        {"address": "<SIGNER>", "writable": true, "signer": true},
        {"address": "<ADDRESS_A>", "writable": true},
        {"address": "<ADDRESS_B>", "writable": true},
        {"address": "<POOL_ADDRESS>", "writable": true},
        {"address": "<VAULT_A>", "writable": true},
        {"address": "<VAULT_B>", "writable": true}
      ],
      "data": {"data": "<ENCODED_SWAP_INSTRUCTION>", "encoding": "base64"}
    },
    "amountOffset": 8,
    "taker": {"user": [0], "inputTokenAccount": 1, "outputTokenAccount": 2}
  }
]
```



</details>

This shows whether the venue would've won on price but not whether a router would've actually routed the swap to it.

It's typically used when the DEX isn't yet integrated with any aggregators yet, but it's also useful for comparing the competitiveness of a venue's prices independent of aggregator routing decisions. In other words, it can be thought of as _"true reality"_ instead of _"router's view of reality"_.
