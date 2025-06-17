# Indexing Data

### Background

The indexer service extends the Data Anchor’s on-chain system to provide structured, long-term storage. It streams on-chain events and stores data blobs for efficient retrieval. By abstracting away low-level details like transaction signatures and slot numbers, it allows developers to work with higher-level, context-aware queries.

Rather than querying raw data such as “What happened in transaction X at slot 34,594?”, users can ask more intuitive questions like “How many proofs did Alice’s node submit in June?”

### Naming Convention

To use the indexer effectively, users have to follow a predefined naming convention. This scheme isn’t enforced programmatically, but it’s required for the indexer to parse and categorize events correctly.

If a user’s account should be associated with a specific network, its namespace should conform to the pattern <mark style="color:red;">`<user>.<network>`</mark>—where the user name is a unique identifier within the network and the network functions like a domain. If a user can own multiple nodes in the same network, each one should set the user as a subdomain, such as:

* <mark style="color:red;">`<account_1>.<user>.<network>`</mark> and <mark style="color:red;">`<account_2>.<user>.<network>`</mark>.

For example, in a network called “Cloud Network” where users can run multiple nodes, a contributor named Alice might set up one node from home and another from the office. In this case, her namespaces would be:

* <mark style="color:red;">`home.alice.cloud`</mark>  and <mark style="color:red;">`office.alice.cloud`</mark>, respectively.

This naming structure allows the indexer to group events by network and namespace, which allows for more intuitive queries and meaningful insights.

### API

Typically, each user account is represented by a _blober_, and each _blober’s_ associated data are its _blobs_. However, a _blober_ may map to a user account, a node account, or another logical entity—the semantics may vary and is determined by each network’s operators.

### Usage

```bash
curl "<INDEXER-URL>" -XPOST \\\\
    -H 'Content-Type: application/json' \\\\
    --data '{"jsonrpc":"2.0","id":1,"method":"get_blobs","params":["BAugq2PZwXBCw72YTRe93kgw3X6ghB3HfF7eSYBDhTsK",385430344]}'
```
