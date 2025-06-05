# Data Flow

#### Under the Hood

<figure><img src="../../../.gitbook/assets/Termina Diagrams.png" alt=""><figcaption></figcaption></figure>

The Data Anchor leverages a modified version of state compression.

For each dataset, the module hashes individual elements and generates a commitment over the whole set. This commitment is saved in Solana’s accounts space, while a compressed form of the original data is stored in Solana’s ledger history. If longer term storage is needed beyond the ledger’s short lifespan of a few days, the module supports external storage providers such as IPFS, Light Protocol’s archival nodes, or self-hosted cloud databases.

It’s easy to check the presence or absence of any data element by submitting a Merkle proof on-chain. The data is temporary and can be removed after a specified period or event, e.g. X time has elapsed or Y event has occurred, which puts a maximum time bound on the elements. Since Solana requires rent for accounts, removing outdated data allows rent to be reclaimed and prevents the lock-up of increasing amounts of capital over time.

To help teams plan their integration, here’s a snapshot of current upload performance and cost across supported Solana RPCs:

<table><thead><tr><th width="120.11004638671875">RPC Endpoint </th><th width="173.98101806640625">Data Bandwidth (KB/s)</th><th width="264.192626953125">Cost per GB directly on Solana (SOL)</th><th>Constant with DA (SOL)</th></tr></thead><tbody><tr><td><strong>devnet</strong></td><td>80.2</td><td>7K</td><td>0.001</td></tr><tr><td><strong>mainnet</strong></td><td>145</td><td>7K</td><td>0.001</td></tr><tr><td><strong>mainnet-staked</strong></td><td>150</td><td>7K</td><td>0.001</td></tr></tbody></table>

To achieve 100 KB/s, uploads are priced at a flat 0.006 SOL (6 million lamports) per MB, with no change based on priority fees or network congestion.

***

#### Glossary

| Term      | Definition                                                                                                                         | Notes                                                         |
| --------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Namespace | A single data collection, dependent on the domain                                                                                  | e.g. Alice’s rollup, Bob’s rewards                            |
| Blober    | Generates Merkle proofs over incoming data and is unique to each namespace                                                         |                                                               |
| Blob      | Each logical unit within a namespace or data collection                                                                            | e.g. Alice’s rollup has a blob for every rollup block         |
| Chunk     | A blob may be split into multiple chunks as part of the upload process to Solana so that each transaction stays within size limits | (Internal implementation detail, irrelevant for the end user) |
