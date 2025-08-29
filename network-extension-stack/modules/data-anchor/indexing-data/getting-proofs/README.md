# Getting Proofs

When your application retrieves data from the indexer, each blob and account change comes with a built-in [data-anchor-proof](https://crates.io/search?q=data-anchor-proofs) you can verify yourself.

These proofs tie every piece of fetched information back to exactly what happened on-chain, so you never have to take anything on faith.

To verify ll you need is a one-time dependency and a few lines of code. Supply the expected on-chain hashes (from standard Solana RPC calls or your trusted source), feed each proof its inputs, and call `verify`.

If any step fails, you’ll immediately know where the mismatch occurred.

<table><thead><tr><th width="224.228515625">RPC Call</th><th>Description</th></tr></thead><tbody><tr><td><a href="https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/indexing-data/getting-proofs/get_proof_for_blob"><code>get_proof_for_blob</code></a></td><td>Returns a <code>CompoundProof</code> bundle for one specific blob account.</td></tr><tr><td><a href="https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/indexing-data/getting-proofs/get_proof"><code>get_proof</code></a></td><td>Returns a full <code>CompoundProof</code> for every blob in a slot—covering inclusion, bank/slot hashes, etc.</td></tr></tbody></table>
