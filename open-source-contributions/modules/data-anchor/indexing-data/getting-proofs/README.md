# Getting Proofs

When your application retrieves data from the indexer, each blob and account change comes with a built-in ZK proof that can be verified directly on-chain.

These proofs tie every piece of fetched information back to exactly what happened in the ledger history, so no data will ever be taken on faith.

<table><thead><tr><th width="224.228515625">RPC Call</th><th>Description</th></tr></thead><tbody><tr><td><a href="checkpoint_proof.md">checkpoint_proof</a></td><td>Initiate a ZK proof committing to a series of data blobs.</td></tr><tr><td><a href="get_proof_request_status.md">get_proof_request_status</a></td><td>Returns the current status of the ZK proof generation: created, completed, failed, etc.</td></tr></tbody></table>
