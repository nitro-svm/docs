# DirectFill Workflow Example

**What you get:** for every historical swap on your pair in the chosen slot range, the sim runs the same trade through your pool, using your price at that moment, and compares your output with the fill the trader actually got on-chain. The result is a per-trade and average bps comparison.

**What you need:**

* Your pool's program. Nothing to supply, since it's on mainnet and the sim loads it automatically.
* Your pair has to already trade somewhere on mainnet, so there's historical flow to test against.
* Your price list with timestamps.

#### Step 1: Pick your pair and slot range

```
sim ranges --api-key <KEY> --after <date>
```

Pick a range where your pair trades actively. Shorter ranges, hours rather than days, are easier to start with.

#### Step 2: Build your pool accounts

You don't need to write Pool's account layout from scratch. Copy an existing pool and edit it:

1. Fetch an existing pool's account data with `getAccountInfo`. This is your model.
2. Change these fields:
   * **Mints:** your pair.
   * **Vaults:** the token accounts holding your inventory. Create one for each side, with balances set to the inventory you want to test with.
   * **Price fields:** the first price in your list.
3. Keep the owner as Pool's program, and keep the account rent-exempt.

Each account is given in this format:

```json
"<pool address>": {
  "data": { "data": "<base64 account data>", "encoding": "base64" },
  "owner": "<Pool program ID>",
  "lamports": <rent-exempt amount>,
  "executable": false,
  "space": <data length>
}
```

You can use a new address or reuse an existing pool's address. For direct fill it makes no difference.

Note: if Pool checks relationships between accounts, such as vaults being derived from the pool and mint, those have to be consistent, or swaps will fail.

#### Step 3: Turn your price list into a schedule

Each price point becomes an account override at a slot:

| Slot | What's written                                 |
| ---- | ---------------------------------------------- |
| 1000 | pool + both vaults (initial state, price 1.00) |
| 1003 | pool with price 1.01                           |
| 1007 | pool with price 0.99                           |

* An override takes effect at the start of its slot and stays until a later override replaces that account. You only include a slot when your price changes.
* After the first slot, you only need to resend the pool account (the one holding the price), not the vaults.
* To convert timestamps to slots: a slot is about 400ms. For precise mapping, use `getBlockTime` on a slot near the start of your range as an anchor.

#### Step 4: Write the swap template

One entry per direction. For example, `USDC → TOKEN` and `TOKEN → USDC` are two entries:

```json
[
  {
    "label": "pool/my-pool/usdc-to-token",
    "mode": "ExactIn",
    "inputMint": "<USDC>",
    "outputMint": "<your token>",
    "instruction": {
      "programId": "<Pool program ID>",
      "accounts": [
        {"address": "11111111111111111111111111111111", "writable": true, "signer": true},
        {"address": "11111111111111111111111111111111", "writable": true},
        {"address": "11111111111111111111111111111111", "writable": true},
        {"address": "<your pool>", "writable": true},
        {"address": "<your vault A>", "writable": true},
        {"address": "<your vault B>", "writable": true}
      ],
      "data": {"data": "<base64 swap instruction data>", "encoding": "base64"}
    },
    "amountOffset": 8,
    "taker": {"user": [0], "inputTokenAccount": 1, "outputTokenAccount": 2}
  }
]
```

* Use Pool's swap instruction in its exact account order. Copying a real Pool swap transaction is the easiest way.
* Put the placeholder `1111…1111` in the trader positions, and list those positions under `taker`. The sim swaps in each historical trader.
* `amountOffset` is the byte position of the u64 amount in the instruction data. The sim writes each trade's size there.
* **Set minimum output to 0** in the instruction data, or trades will be rejected.

#### Step 5: Run it

Then run it in a sim CLI

```bash
sim run --api-key <KEY> \ 
--start-slot <START> --end-slot <END> \ 
--reroute-order-flow venue \ 
--venue-templates templates.json \ 
--reroute-aggregators jupiter,okx,titan,dflow
```
