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

### RPC Endpoints

<table><thead><tr><th width="175.30859375">Network Configuration</th><th>RPC Endpoint</th></tr></thead><tbody><tr><td><strong>Devnet</strong></td><td><code>https://devnet.indexer.data-anchor.termina.technology/</code></td></tr><tr><td><strong>Mainnet</strong></td><td><code>https://mainnet.indexer.data-anchor.termina.technology/</code></td></tr></tbody></table>

### Usage

{% hint style="info" %}
**API key is optional.** If you need one, [reach out](http://t.me/oceanicursula) to the team to get access.
{% endhint %}

```bash
curl "<INDEXER-URL>"
  -X POST \
  -H 'Content-Type: application/json' \
  -H "x-api-key: <YOUR_API_KEY>" \
  -d '{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "get_blobs",
    "params": [
      "BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",
      385430344
    ]
  }'
```
