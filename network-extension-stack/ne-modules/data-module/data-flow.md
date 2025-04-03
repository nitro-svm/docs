# Data Flow

#### Under the Hood

<figure><img src="../../../.gitbook/assets/Termina Diagrams (5) (2).png" alt=""><figcaption></figcaption></figure>

The data module leverages a modified version of state compression.

For each dataset, the module hashes individual elements and generates a commitment over the whole set. This commitment is saved in Solana’s accounts space, while a compressed form of the original data is stored in Solana’s ledger history. If longer term storage is needed beyond the ledger’s short lifespan of a few days, the module supports external storage providers such as IPFS, Light Protocol’s archival nodes, or self-hosted cloud databases.

It’s easy to check the presence or absence of any data element by submitting a Merkle proof on-chain. The data is temporary and can be removed after a specified period or event, e.g. X time has elapsed or Y event has occurred, which puts a maximum time bound on the elements. Since Solana requires rent for accounts, removing outdated data allows rent to be reclaimed and prevents the lock-up of increasing amounts of capital over time.

#### Glossary

* Namespace: describes a data collection, dependent on the domain e.g. Alice’s rollup, Bob’s rewards.
* Blober: responsible for generating Merkle proofs over incoming data and is unique to each namespace.
* Blob: describes each logical unit within a data collection e.g. Alice’s rollup has a blob for every rollup block.
* Chunk: blob may be split into chunks to be uploaded to Solana so that each transaction stays within size limits. (Internal implementation detail, irrelevant for the end user.)
