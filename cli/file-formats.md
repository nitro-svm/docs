# File Formats

`sim run` reads two JSON files:

* [`--account-overrides <FILE>`](#account-overrides): account states that replace the onchain ones.
* [`--venue-templates <FILE>`](#venue-templates): the instructions that price swaps directly on your venue.

`sim` checks both files before it starts a session. A bad file stops the run with a usage error (exit status 2) that names the file and the problem.

## Account Overrides

A JSON object keyed by base58 address. Each value is the account's full state at `--start-slot`.

```json
{
  "<ADDRESS>": {
    "lamports": 2039280,
    "owner": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA",
    "data": ["<BASE64_BYTES>", "base64"],
    "executable": false,
    "space": 165
  }
}
```

| Field | Type | Meaning |
| --- | --- | --- |
| `lamports` | integer | Balance in lamports. 1 SOL = 1,000,000,000 lamports. |
| `owner` | base58 string | The program that owns the account. |
| `data` | `["<base64>", "base64"]` | The account's bytes. `{"data": "<base64>", "encoding": "base64"}` also works. |
| `executable` | boolean | Must be `false`. |
| `space` | integer | The length of `data` in bytes. The account's size comes from `data`, so keep the two equal. |

* An override replaces the whole account. To change a token balance, write the whole token account: 165 bytes in the SPL token layout. Rust users can build it with [`simulator_api::token`](https://docs.rs/simulator-api/latest/simulator_api/token/index.html).
* The range's transactions run on top of the overrides. With `--reroute-order-flow`, rerouting sees the file's state for the whole run.
* Other fields, such as `rentEpoch`, are ignored. So you can start from the `value` that Solana's `getAccountInfo` returns with `"encoding": "base64"`:

```bash
curl -s https://api.mainnet-beta.solana.com -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getAccountInfo","params":["<ADDRESS>",{"encoding":"base64"}]}' \
  | jq '{"<ADDRESS>": .result.value}' > overrides.json
```

That is the account's current state. Edit it to the state you want before the run.

`sim` refuses:

* A file with no accounts.
* An executable account, or one owned by a program loader. To replace a program, use `--program-so`.
* `data` that isn't valid base64.

A pool created in the overrides file can be routed through with `--reroute-extra-markets <POOL>`. See [Rerouting via an Aggregator](../flows/rerouting-historical-flow/rerouting-via-an-aggregator.md).

## Venue Templates

A JSON array of templates. Each template is one instruction that swaps on your venue, plus where `sim` writes the amount and the taker. Use them with `--reroute-order-flow venue`. See [Rerouting Directly to a Venue](../flows/rerouting-historical-flow/rerouting-directly-to-a-venue.md).

```json
[
  {
    "label": "my-pool/usdc-to-token",
    "mode": "ExactIn",
    "inputMint": "<USDC_MINT>",
    "outputMint": "<TOKEN_MINT>",
    "instruction": {
      "programId": "<PROGRAM_ID>",
      "accounts": [
        { "address": "11111111111111111111111111111111", "writable": true, "signer": true },
        { "address": "11111111111111111111111111111111", "writable": true },
        { "address": "11111111111111111111111111111111", "writable": true },
        { "address": "<POOL>", "writable": true },
        { "address": "<VAULT_A>", "writable": true },
        { "address": "<VAULT_B>", "writable": true },
        { "address": "TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA" }
      ],
      "data": { "data": "<BASE64_INSTRUCTION_DATA>", "encoding": "base64" }
    },
    "amountOffset": 8,
    "taker": { "user": [0], "inputTokenAccount": 1, "outputTokenAccount": 2 }
  }
]
```

### Fields

| Field | Meaning |
| --- | --- |
| `label` | The template's name on every result row and summary line. Unique within the file, not blank. |
| `mode` | `ExactIn` or `ExactOut`. The template prices original swaps of this mode only. |
| `inputMint`, `outputMint` | The mints the swap sells and buys. Each direction needs its own template. |
| `instruction.programId` | Your venue's program. |
| `instruction.accounts` | The instruction's accounts, in order: `address`, `writable` and `signer`. The flags default to `false` and are sent as written. 1 to 255 accounts. |
| `instruction.data` | The instruction data, base64: `{"data": "<base64>", "encoding": "base64"}`. |
| `amountOffset` | Byte offset in the data where `sim` writes the swap amount, as a little-endian u64. The 8 bytes must fit in the data. |
| `taker` | Which accounts `sim` sets to the original swap's taker, as indices into `instruction.accounts`. See below. |

Each role takes a list of indices, so you can cover an account the instruction names more than once, or a bare index such as `0`.

| Role | Set to | Rule |
| --- | --- | --- |
| `user` | The original swap's fee payer. | Each account must be a signer. |
| `inputTokenAccount` | The fee payer's associated token account for `inputMint`, funded with the trade. | Each account must be writable. |
| `outputTokenAccount` | The fee payer's associated token account for `outputMint`, emptied first so what lands there is the fill. | Each account must be writable. |
| `derived` (optional) | Addresses your program derives from the taker. | See below. |

The addresses in the `taker` slots are placeholders, since `sim` replaces them. By convention, use `11111111111111111111111111111111`. No two roles can name the same index.

`derived` goes inside `taker`. It is a list of program-derived addresses, each recomputed for every taker:

```json
"taker": {
  "user": [0],
  "inputTokenAccount": 1,
  "outputTokenAccount": 2,
  "derived": [
    { "program": "<PROGRAM_ID>", "seeds": [{ "utf8": "position" }, "user"], "slots": [5, 9] }
  ]
}
```

* `program`: the program the address is derived under.
* `seeds`: 1 to 16 seeds of at most 32 bytes each. `{"utf8": "<text>"}` is a literal string; `"user"` is the taker's address.
* `slots`: the indices that receive the derived address.

### How a Template Is Used

For every hop of an original swap with the same `mode`, `inputMint` and `outputMint`, from the aggregators in `--reroute-aggregators`, `sim`:

1. Writes the hop's amount at `amountOffset`: the input amount for `ExactIn`, the output amount for `ExactOut`.
2. Sets the `taker` accounts to the original swap's fee payer and its token accounts.
3. Simulates the instruction at the original swap's [pricing point](../flows/rerouting-historical-flow/README.md#interpreting-the-numbers). The probe never lands: the original transaction still runs as it did onchain.
4. Compares the fill with the original swap's: the output received for `ExactIn`, the input charged for `ExactOut`. Positive basis points mean the template beat the original swap.

Every template that matches a hop is probed. A probe whose measured token account is gone or unreadable afterwards counts as `unattributed`.

Read the results in the run's [reroute summary and SQLite file](output-format.md#reroute-results).

### Errors

`sim` refuses a template file when:

* It isn't a JSON array, or the array is empty.
* Two templates share a `label`, or a `label` is blank.
* `inputMint` equals `outputMint`.
* An address isn't valid base58. The error names the field, such as `instruction.accounts[2].address`.
* `instruction.data` isn't valid base64, or the amount at `amountOffset` runs past its end.
* A `taker` index is past the last account, or two roles name the same index.
* A `user` account isn't a signer, or a token account isn't writable.
* A `derived` entry has no slots, no seeds, more than 16 seeds or a seed over 32 bytes.

Rust users can find the templates for the venues the client covers in [`simulator_client::recipes::VENUES`](https://docs.rs/simulator-client/latest/simulator_client/recipes/constant.VENUES.html).
