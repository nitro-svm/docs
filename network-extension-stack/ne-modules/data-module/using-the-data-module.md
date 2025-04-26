# Using the Data Module

You can use the Data Module's CLI tool to post data on-chain and retrieve it from the ledger or an indexer service.

{% stepper %}
{% step %}
#### Prerequisite Setup

{% hint style="info" %}
We do not recommend using public RPC nodes for production because of their low rate limits and low stake, which makes transaction landing slower than private RPC nodes. Please reach out if you'd like help in setting up a private endpoint.
{% endhint %}

{% tabs %}
{% tab title="Rust SDK" %}
```sh
cargo add nitro-da-client
```
{% endtab %}

{% tab title="CLI Utility" %}
To interact with on-chain programs, set up Solana CLI:

* Install `solana-cli` by following the [official documentation](https://solana.com/docs/intro/installation).
* Configure `solana-cli` to use your preferred RPC and network e.g.`solana config set --url https://api.devnet.solana.com` for devnet.

Once that's complete, set up the Data Module CLI:

```bash
curl -sSf https://nitro-da-cli.termina.technology/install.sh | sh
```

Run `nitro-da-cli` to see all the available commands and their options.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
#### Select Blober Program ID

Make sure to use the correct program ID for mainnet and devnet:

| Network        | Program ID                                   |
| -------------- | -------------------------------------------- |
| Solana Mainnet | 9i2MEc7s38jLGoEkbFszuTJCL1w3Uorg7qjPjfN8Tv5Z |
| Solana Devnet  | 2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH |
{% endstep %}

{% step %}
#### Create Namespace

A namespace represents a data collection, and only your keypair is allowed to write data to it. To create a sample namespace called `nitro` on Solana devnet:

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let blober_client = BloberClient::builder()
    .payer(payer)
    .program_id(program_id)
    .indexer_from_url(&indexer_url)
    .await?
    .build_with_config(config)
    .await?;
```

* The `payer` is an `Arc<Keypair>` that points to the Solana keypair to use with the client
* The `program_id` is the address of the on-chain blober program to interact with
* The `indexer_url` is an optional parameter if using our indexer service
* The `config` is a `solana_cli_config::Config` object that sets the transaction RPC details
{% endtab %}

{% tab title="CLI Utility" %}
```bash
nitro-da-cli \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blober initialize
```
{% endtab %}
{% endtabs %}

If you no longer need the data commitment, close the namespace and reclaim the rent from the on-chain account:

```bash
nitro-da-cli \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blober close
```
{% endstep %}

{% step %}
#### **Upload Data**

To upload data, store it in a file and point the CLI to that file:

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let transaction_outcomes = blober_client
    .upload_blob(data, fee, blober_id, timeout)
    .await?;
```

* The `data` is a slice of bytes (`&[u8]`)
* The `fee` is a fee strategy that determines the priority fee
* The `blober_id` is the blober PDA, or namespace, the data should be uploaded to
* \[Optional] The `timeout` specifies how long the client should wait before stopping a data upload attempt (if unset, the client will retry indefinitely)
{% endtab %}

{% tab title="CLI Utility" %}
```bash
nitro-da-cli \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blob upload \
    --data-path "./data.txt"
```

Alternatively, you can directly pipe the data into the command or pass it in as a hex string via the `--data` param:

```bash
cat "./data.txt" | nitro-da-cli --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" --namespace "nitro" blob upload
```

```bash
nitro-da-cli \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blob upload \
    --data "<some-hex-string>"
```
{% endtab %}
{% endtabs %}

After the upload is complete, you'll see output similar to the following:

```bash
Slot: 3323, Signatures: [2Vkwid2ZTman9zvGq2Tp7P2S5CUWJ2fsZSQzGzaoFkDMAgS9xNccCvNe7PJuHrXNotsVu3BoJAsRa9jdfbZraXvS, faRmYWXPQUDJcFpqffJVE49f5aMSCLYnqp1xH3DZc3SM2Uayc7jReRfR6LjNkFxeuviSJTXMTtSAmAL9tAppwyK, 5XsiKe95nk9GmkceXQkEWapAuAcEQFFYEqMfh5kyeszSxjXepSyDbgzEXmzoQniWMdWvv6mVm5Qbyh9e1i8hHF7K, 2nk2Fj2xwM7oRfsqbmDBNqYwPCLGxHcGUujBo7napJgavMSWFEQ6C9wYmLCkKcuaetBs89vtMbtYzEKaKLKjasKd, 43zzTdgoZBR3sDphuPQTZQHZdT4Ms976bRiY8jguHPZbNPibY3k4EVnrRGKCbUy97i1RzsdMRXkYyv2KJZp9MQZE], Success: true
```
{% endstep %}

{% step %}
#### **Fetch Data from Ledger**

Use the signatures from the previous step to retrieve the original data from Solana:

{% tabs %}
{% tab title="Rust SDK" %}
<pre class="language-rust"><code class="lang-rust"><strong>let blob = blober_client
</strong>    .get_ledger_blobs_from_signatures(blober, signatures)
    .await?;
</code></pre>

Alternatively, set the slot number and number of lookback slots to fetch historical data.

```rust
let blobs = blober_client
    .get_ledger_blobs(slot, blober, lookback_slots)
    .await?;
```
{% endtab %}

{% tab title="CLI Utility" %}
```bash
nitro-da-cli \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blob fetch 2Vkwid2ZTman9zvGq2Tp7P2S5CUWJ2fsZSQzGzaoFkDMAgS9xNccCvNe7PJuHrXNotsVu3BoJAsRa9jdfbZraXvS faRmYWXPQUDJcFpqffJVE49f5aMSCLYnqp1xH3DZc3SM2Uayc7jReRfR6LjNkFxeuviSJTXMTtSAmAL9tAppwyK 5XsiKe95nk9GmkceXQkEWapAuAcEQFFYEqMfh5kyeszSxjXepSyDbgzEXmzoQniWMdWvv6mVm5Qbyh9e1i8hHF7K 2nk2Fj2xwM7oRfsqbmDBNqYwPCLGxHcGUujBo7napJgavMSWFEQ6C9wYmLCkKcuaetBs89vtMbtYzEKaKLKjasKd 43zzTdgoZBR3sDphuPQTZQHZdT4Ms976bRiY8jguHPZbNPibY3k4EVnrRGKCbUy97i1RzsdMRXkYyv2KJZp9MQZE
```
{% endtab %}
{% endtabs %}

This returns hex encoded bytes, so remember to hex decode it to get the data back in its original form.
{% endstep %}

{% step %}
#### Fetch Data from Indexer (Optional)

If you need the data to be stored for a longer time period than the ledger's lifetime of three days, you may want to use an indexer.

{% hint style="info" %}
Please reach out if you'd like us to set up an indexer on your behalf.
{% endhint %}

To retrieve data from the indexer, all you need to provide is the slot number at which the blob was finalized. (You can find this value from the upload command above.)

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let blobs = blober_client
    .get_blobs(slot, blober)
    .await?;
```
{% endtab %}

{% tab title="CLI Utility" %}
```bash
nitro-da-cli \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    --indexer-url "wss://your-indexer.deployment" \
    indexer blobs 3323
```
{% endtab %}
{% endtabs %}

This will produce an output similar to the one from fetching data from the ledger.

In addition, you can also request a proof of inclusion or proof of completeness from the indexer:

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let proof = blober_client
    .get_slot_proof(slot, blober)
    .await?;
```
{% endtab %}

{% tab title="CLI Utility" %}
```bash
nitro-da-cli \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    --indexer-url "wss://your-indexer.deployment" \
    indexer proofs 3323
```
{% endtab %}
{% endtabs %}

This contains metadata that prove that the indexer's return value is correct, and it hasn’t tampered with the original data posted on-chain.
{% endstep %}
{% endstepper %}
