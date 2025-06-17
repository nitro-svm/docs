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

To retrieve data from the indexer, all you need to provide is the slot number at which the blob was finalized. (You can find this value from the upload command above.)

This will produce an output similar to the one from fetching data from the ledger.

In addition, you can also request a proof of inclusion or proof of completeness from the indexer:

### API Reference

Under the hood, the indexer exposes data via a <mark style="color:orange;">`JSON-RPC`</mark> server.&#x20;

<details>

<summary>get_blobs</summary>

Retrieve a list of blobs for a given slot & blober pubkey.&#x20;

It returns an error if there was a database or RPC failure, and None if the slot has not been completed yet.&#x20;

If the slot is completed but no blobs were uploaded, an empty list will be returned.

**Signature**

```rust
async fn get_blobs(&self, blober: Pubkey, slot: u64) -> RpcResult<Option<Vec<Vec<u8>>>>;w
```

</details>

<details>

<summary><strong>get_blobs_by_blober</strong></summary>

Retrieve a list of blobs for a given blober pubkey and time range.&#x20;

Returns an error if there was a database or RPC failure, and an empty list if no blobs were found.

**Signature**

```rust
async fn get_blobs_by_blober(&self, blober: BlobsByBlober) -> RpcResult<Vec<Vec<u8>>>;
```

**Parameters**:

```rust
pub struct BlobsByBlober {
    pub blober: Pubkey,
    #[serde(flatten)]
    pub time_range: TimeRange,
}
pub struct TimeRange {
    pub start: Option<DateTime<Utc>>,
    pub end:   Option<DateTime<Utc>>,
}
```

Which equals to the following options in JSON:

```json
{
  "blober": "BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK"
}
```

```json
{
  "blober": "BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",
  "start": "2025-06-09T14:35:06.538958843Z"
}
```

```json
{
  "blober": "BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",
  "start": "2025-06-09T14:30:06.538958843Z",
  "end": "2025-06-09T14:35:06.538958843Z"
}
```

</details>

<details>

<summary><strong>get_blobs_by_payer</strong></summary>

Retrieve a list of blobs for a given payer pubkey, network ID, and time range. Returns an error if there was a database or RPC failure, and an empty list if no blobs were found.

#### Signature

```rust
async fn get_blobs_by_payer(&self, payer: BlobsByPayer) -> RpcResult<Vec<Vec<u8>>>;
```

**Parameters**

```rust
pub struct BlobsByPayer {
    pub payer: Pubkey,
    pub network_name: String,
    #[serde(flatten)]
    pub time_range: TimeRange,
}
```

Which equals to the following options in JSON:

```json
{
  "payer": "BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",
  "network_name": "ping"
}
```

```json
{
  "payer": "BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",
  "network_name": "ping",
  "start": "2025-06-09T14:35:06.538958843Z"
}
```

```json
{
  "payer": "BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",
  "network_name": "ping",
  "start": "2025-06-09T14:30:06.538958843Z",
  "end": "2025-06-09T14:35:06.538958843Z"
}
```

</details>

<details>

<summary>get_proof</summary>

Retrieve a proof for a given slot and blober pubkey. Returns an error if there was a database or RPC failure, and None if the slot has not been completed yet.

#### Signature

```rust
async fn get_proof(&self, blober: Pubkey, slot: u64) -> RpcResult<Option<CompoundProof>>;
```

</details>

<details>

<summary>get_proof_for_blob</summary>

Retrieve a compound proof that covers a particular blob. Returns an error if there was a database or RPC failure, and None if the blob does not exist.

#### Signature

```rust
async fn get_proof_for_blob(&self, blob_address: Pubkey) -> RpcResult<Option<CompoundProof>>;
```

### Usage

To use the indexer API you can either use our pre-built CLI, Rust client or if calling from any other language, simply create and send `JSONRPC` requests.

#### curl example

```bash
curl "<INDEXER-URL>" -XPOST \\\\
    -H 'Content-Type: application/json' \\\\
    --data '{"jsonrpc":"2.0","id":1,"method":"get_blobs","params":["BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",385430344]}'
```

</details>

### **Data Anchor Proofs**

The Data Anchor client-side crate lets you verify data returned by the indexer against on-chain state.&#x20;

It exposes these proof types:

1. **Accounts Delta Hash Proofs**\
   Prove inclusion or exclusion of an account in the slot’s updated-accounts Merkle tree. The root (<mark style="color:orange;">`accounts_delta_hash`</mark>) is checked against a <mark style="color:orange;">`BankHashProof`</mark>.
2. **Bank Hash Proofs**\
   Hash together parent bank hash, accounts delta hash, signature count, and blockhash to reproduce the bank hash returned by Solana RPC.
3. **Slot Hash Proofs**\
   Verify that the bank hash was recorded in the <mark style="color:orange;">`SlotHashes`</mark> sysvar for a specific slot.
4. **Blob Proofs**\
   Confirm that a given blob’s raw bytes hash to the expected digest stored on-chain.
5. **Compound Proofs**\
   Combine the above into one structure—either <mark style="color:orange;">`CompoundInclusionProof`</mark> (blobs present) or <mark style="color:orange;">`CompoundCompletenessProof`</mark> (blobs absent) for a slot or blob address.

***

**Verifying Proofs**

```rust
// Accounts delta hash
use data_anchor_proofs::accounts_delta_hash::{
    inclusion::InclusionProof, 
    exclusion::ExclusionProof
};
inclusion_proof.verify(expected_accounts_delta_hash);
exclusion_proof.verify(expected_accounts_delta_hash)?;

// Bank hash
use data_anchor_proofs::bank_hash::BankHashProof;
bank_hash_proof.verify(expected_bank_hash);

// Slot hash
use data_anchor_proofs::slot_hash::SlotHashProof;
slot_hash_proof.verify(slot, bank_hash, accounts_delta_hash)?;

// Blob proof
use data_anchor_proofs::blob::BlobProof;
blob_proof.verify(&blob_bytes)?;

// Compound proofs
use data_anchor_proofs::compound::{
    inclusion::CompoundInclusionProof,
    completeness::CompoundCompletenessProof,
    inclusion::ProofBlob
};
compound_proof.verify(
    blober_program,
    blockhash,
    &[ProofBlob { blob: blob_pubkey, data: Some(blob_bytes) }]
)?;
```

