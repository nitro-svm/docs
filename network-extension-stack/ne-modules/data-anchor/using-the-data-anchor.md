# Using the Data Anchor

You can use the Data Anchor's Rust client or CLI tool to post data on-chain and retrieve it from the ledger or our indexer service.

{% hint style="info" %}
We do not recommend using public RPC nodes for production because of their low rate limits and low stake, which makes transaction landing slower than private RPC nodes.&#x20;

Please reach out if you'd like help in setting up a private endpoint.
{% endhint %}

{% stepper %}
{% step %}
#### Prerequisite Setup

If your codebase is in Rust, please use the Rust client. Otherwise, we also provide a command line utility.

{% tabs %}
{% tab title="Rust SDK" %}
Install it directly:

<pre class="language-sh"><code class="lang-sh"><strong>cargo add data-anchor-client
</strong></code></pre>

Or add the latest version in your Cargo.toml:

{% code title="Cargo.toml" %}
```toml
[dependencies]
data-anchor-client = "0.1.x"
```
{% endcode %}
{% endtab %}

{% tab title="CLI Utility" %}
* Install `solana-cli` by following the [official documentation](https://solana.com/docs/intro/installation).

```sh
curl --proto '=https' --tlsv1.2 -sSfL https://solana-install.solana.workers.dev | bash
```

* Configure `solana-cli` to use your preferred RPC and network e.g.`solana config set --url https://api.devnet.solana.com` for devnet.
* Download `data-anchor`.

```bash
curl -sSf https://data-anchor.termina.technology/install.sh | sh
```

* Run `data-anchor` to see all the available commands and their options.
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
data-anchor \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blober initialize
```
{% endtab %}
{% endtabs %}

If you no longer need the data commitment, close the namespace and reclaim the rent from the on-chain account:

```bash
data-anchor \
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
data-anchor \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blob upload \
    --data-path "./data.txt"
```

Alternatively, you can directly pipe the data into the command or pass it in as a hex string via the `--data` param:

```bash
cat "./data.txt" | data-anchor --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" --namespace "nitro" blob upload
```

```bash
data-anchor \
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
data-anchor \
    --program-id "2RWsr92iL39YCLiZu7dZ5hron4oexEMbgWDg35v5U5tH" \
    --namespace "nitro" \
    blob fetch 2Vkwid2ZTman9zvGq2Tp7P2S5CUWJ2fsZSQzGzaoFkDMAgS9xNccCvNe7PJuHrXNotsVu3BoJAsRa9jdfbZraXvS faRmYWXPQUDJcFpqffJVE49f5aMSCLYnqp1xH3DZc3SM2Uayc7jReRfR6LjNkFxeuviSJTXMTtSAmAL9tAppwyK 5XsiKe95nk9GmkceXQkEWapAuAcEQFFYEqMfh5kyeszSxjXepSyDbgzEXmzoQniWMdWvv6mVm5Qbyh9e1i8hHF7K 2nk2Fj2xwM7oRfsqbmDBNqYwPCLGxHcGUujBo7napJgavMSWFEQ6C9wYmLCkKcuaetBs89vtMbtYzEKaKLKjasKd 43zzTdgoZBR3sDphuPQTZQHZdT4Ms976bRiY8jguHPZbNPibY3k4EVnrRGKCbUy97i1RzsdMRXkYyv2KJZp9MQZE
```
{% endtab %}
{% endtabs %}

This returns hex encoded bytes, so remember to hex decode it to get the data back in its original form.
{% endstep %}
{% endstepper %}

### Indexer Services

{% hint style="info" %}
Please reach out if you'd like us to set up an indexer on your behalf.
{% endhint %}

If you need the data to be stored for a longer time period than the ledger's lifetime of three days, you may want to use an indexer.

To retrieve data from the indexer, all you need to provide is the slot number at which the blob was finalized (You can find this value from the upload command above).

You can follow on the steps in [Indexing Data](https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/indexing-data) to walk through an end-to-end guide of different ways to index data you've uploaded.
