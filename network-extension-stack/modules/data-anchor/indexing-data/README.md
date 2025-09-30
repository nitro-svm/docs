# Indexing Data

### Background

The indexer service extends the Data Anchor’s on-chain system to provide structured, long-term storage. It streams on-chain events and stores data blobs for efficient retrieval. By abstracting away low-level details like transaction signatures and slot numbers, it allows developers to work with higher-level, context-aware queries.

Rather than querying raw data such as “What happened in transaction X at slot 34,594?”, users can ask more intuitive questions like “How many proofs did Alice’s node submit in June?”

### Naming Convention

To use the indexer effectively, users have to follow a predefined naming convention. This scheme isn’t enforced programmatically, but it’s required for the indexer to parse and categorize events correctly.

If a user’s account should be associated with a specific network, its namespace should conform to the pattern `<user>.<network>`—where the user name is a unique identifier within the network and the network functions like a domain. If a user can own multiple nodes in the same network, each one should set the user as a subdomain, such as:

* `<account_1>.<user>.<network>` and `<account_2>.<user>.<network>`.

For example, in a network called “Cloud Network” where users can run multiple nodes, a contributor named Alice might set up one node from home and another from the office. In this case, her namespaces would be:

* `home.alice.cloud`  and `office.alice.cloud`, respectively.

This naming structure allows the indexer to group events by network and namespace, which allows for more intuitive queries and meaningful insights.

### ZK Proofs

The Data Anchor uses a hashchain mechanism to commit to all ledger data in a verifiable way.\
To prove the integrity of a given data blob, we generate a fixed-size ZK proof that attests to an arbitrary range of data. This proof is then submitted onchain, where it can be publicly verified. The verification ensures that the onchain commitment exactly matches the original data and that no tampering has occurred.

### Getting Started

Once you've uploaded data to your blober PDA [using the Data Anchor](https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/using-the-data-anchor), you can use the indexer service to query data in an intuitive way beyond the ledger lifetime.

{% stepper %}
{% step %}
### Configure Indexer Access

Please [reach out](http://t.me/oceanicursula) to the team for an API key to access the indexer's endpoints and use the corresponding URLs for devnet and mainnet.

<table><thead><tr><th width="175.30859375">Network</th><th>RPC Endpoint</th></tr></thead><tbody><tr><td><strong>Devnet</strong></td><td><code>https://devnet.data-anchor.termina.technology/</code></td></tr><tr><td><strong>Mainnet</strong></td><td><code>https://mainnet.data-anchor.termina.technology/</code></td></tr></tbody></table>

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let indexer_url = "https://devnet.data-anchor.termina.technology/";
let data_anchor_client = DataAnchorClient::builder()
    .payer(payer)
    .program_id(program_id)
    .indexer_from_url(&indexer_url)
    .await?
    .build_with_config(config)
    .await?;
```

Add the `indexer_url` to the client builder configuration to enable indexer-related methods.
{% endtab %}

{% tab title="CLI Utility" %}
```bash
INDEXER_URL="https://devnet.indexer.data-anchor.termina.technology/"
```

Set the indexer URL as an environment variable or pass it directly using the `--indexer-url` flag on each command.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
### Query by Slot

Retrieve data uploaded to a specific account in a given slot by passing the account’s namespace and the corresponding slot number.

The namespace is the ASCII identifier used during initialization, and the slot number is returned during the [`upload_blob`](https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/using-the-data-anchor) step.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let slot = your_upload_slot;
let blobs = data_anchor_client
    .get_blobs(slot, namespace.into())
    .await?;
```
{% endtab %}

{% tab title="CLI Utility" %}
```bash
data-anchor \
    --namespace $NAMESPACE \
    --indexer-url $INDEXER_URL \
    indexer blobs <SLOT_NUMBER>
```
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
### Query by Blober PDA

Retrieve all data associated with your blober PDA across different time periods. This provides a complete view of all uploads to your namespace and supports optional time range filtering.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let blobs = data_anchor_client
    .get_blobs_by_blober(namespace.into(), None)
    .await?;
```

To query all blobs for the namespace without time restrictions, use `None` for the time range.

```rust
use chrono::{DateTime, Utc};

let start_time = DateTime::parse_from_rfc3339("2025-06-01T00:00:00Z")?.with_timezone(&Utc);
let end_time = DateTime::parse_from_rfc3339("2025-06-30T00:00:00Z")?.with_timezone(&Utc);
let time_range = Some((start_time, end_time));

let blobs = data_anchor_client
    .get_blobs_by_blober(namespace.into(), time_range)
    .await?;
```

To filter by time, create a time range tuple using RFC3339 timestamps.
{% endtab %}

{% tab title="CLI Utility" %}
```bash
data-anchor \
    --indexer-url $INDEXER_URL \
    indexer blobs-for-blober \
    --blober <BLOBER_PUBKEY>
    --start "2025-06-01T00:00:00Z" \
    --end "2025-06-30T00:00:00Z"
```

Replace `<BLOBER_PUBKEY>` with the blober PDA's public key, which is derived from the account's namespace.

You can also add optional `--start` and `--end` flags to filter by time range. Time range filtering accepts RFC3339 timestamps (e.g., `2025-06-01T00:00:00Z`) and can be applied to most indexer query methods.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
### Query by Network and Namespace

If the namespace follows the recommended [naming convention](https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/indexing-data) (`<user>.<network>`), it's possible to use network-wide queries and user-specific filtering.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let network_name = "your_network";
let network_blobs = data_anchor_client
    .get_blobs_by_network(network_name.to_string(), time_range)
    .await?;
```

Use `get_blobs_by_network` to query all data that users have uploaded across the network.

```rust
let namespace = "user.network";
let payer_pubkey = Some(your_payer_pubkey);
let namespace_blobs = data_anchor_client
    .get_blobs_by_namespace_for_payer(namespace.into(), payer_pubkey, time_range)
    .await?;
```

Use `get_blobs_by_namespace_for_payer` for granular filtering by specific users.&#x20;

The `payer-pubkey` parameter is optional to show only the blobs uploaded by that account in the namespace.
{% endtab %}

{% tab title="CLI Utility" %}
```bash
data-anchor \
    --indexer-url $INDEXER_URL \
    indexer blobs-for-network \
    --network-name <NETWORK_NAME> \
    --start "2025-06-01T00:00:00Z" \
    --end "2025-06-30T00:00:00Z"
```

Use `blobs-for-network` to retrieve all data blobs uploaded across an entire network, optionally within a specified time range.

```bash
data-anchor \
    --indexer-url $INDEXER_URL \
    indexer blobs-for-namespace \
    --namespace <NAMESPACE> \
    --payer-pubkey <PAYER_PUBKEY> \
    --start "2025-06-01T00:00:00Z" \
    --end "2025-06-30T00:00:00Z"
```

Use `blobs-for-namespace` retrieve all data blobs associated with a specific namespace and optionally within a specified time range.

The `--payer-pubkey` parameter is optional to show only the blobs uploaded by that account in the namespace.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
### Query by Payer Address

Retrieve all data uploads paid for by a specific payer address within a network context. This is useful for tracking contributions from specific accounts or analyzing payment patterns.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let payer_pubkey = your_payer_pubkey;
let network_name = "your_network";
let payer_blobs = data_anchor_client
    .get_blobs_by_payer(payer_pubkey, network_name.to_string(), time_range)
    .await?;
```

Query all blobs paid for by a specific payer within a given network, with optional time range filtering.
{% endtab %}

{% tab title="CLI Utility" %}
```bash
data-anchor \
    --indexer-url $INDEXER_URL \
    indexer blobs-for-payer \
    --blob-payer <PAYER_PUBKEY> \
    --network-name <NETWORK_NAME> \
    --start "2025-06-01T00:00:00Z" \
    --end "2025-06-30T00:00:00Z"
```

Use `--blob-payer` to specify the payer address and `--network-name` to scope the query to a specific network context.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
### Retrieve Proofs

The indexer provides cryptographic proofs that verify the authenticity of returned data. These proofs ensure data integrity and prove that the indexer hasn't tampered with the original uploads. See [getting-proofs](getting-proofs/ "mention") for custom proofs.

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let proof_type = CustomerElf::DataCorrectness;
let request_id = data_anchor_client
    .checkpoint_custom_proof(slot, blober_pda, proof_type)
    .await?;
let status = data_anchor_client
    .get_proof_request_status(request_id)
    .await?;
```
{% endtab %}

{% tab title="CLI Utility" %}
```bash
data-anchor \
    --namespace $NAMESPACE \
    --indexer-url $INDEXER_URL \
    indexer zk-proof --slot <SLOT_NUMBER> --proof-type data-correctness
```

Replace `<SLOT_NUMBER>` with the slot number from your upload output. This generates a ZK proof for all blobs since the last checkpoint.
{% endtab %}
{% endtabs %}
{% endstep %}

{% step %}
### Process Blob Metadata

The indexer returns structured data that includes your original blob data along with helpful metadata. Output formats vary between the Rust SDK (structured objects) and CLI utility (configurable text, JSON, or CSV formats).

{% tabs %}
{% tab title="Rust SDK" %}
```rust
let blobs = data_anchor_client.get_blobs(slot, namespace.into()).await?;

for blob in blobs {
    println!("Blob ID: {}", blob.id);
    println!("Slot: {}", blob.slot);
    println!("Timestamp: {}", blob.timestamp);
    println!("Data length: {} bytes", blob.data.len());
    
    // Your original uploaded data is ready to use
    let original_data = blob.data;
    // Process original_data as needed
}
```

The SDK returns structured objects with decoded data and metadata. Access your original data through the `data` field without any additional decoding.
{% endtab %}

{% tab title="CLI Utility" %}
```bash
data-anchor \
    --namespace $NAMESPACE \
    --indexer-url $INDEXER_URL \
    indexer blobs <SLOT_NUMBER>
```

The default text output provides human-readable information about the blobs, including slot numbers, timestamps, and data summaries.

You can also add the `--output "json"` flag for structured output that's easier to parse programmatically, or omit the flag for human-readable text output.
{% endtab %}
{% endtabs %}
{% endstep %}
{% endstepper %}
