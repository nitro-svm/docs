# Getting Proofs

When your application retrieves data from the indexer, each blob and account change comes with a built-in [data-anchor-proof](https://crates.io/search?q=data-anchor-proofs) you can verify yourself.

These proofs tie every piece of fetched information back to exactly what happened on-chain, so you never have to take anything on faith.

To verify ll you need is a one-time dependency and a few lines of code. Supply the expected on-chain hashes (from standard Solana RPC calls or your trusted source), feed each proof its inputs, and call `verify`.

If any step fails, you’ll immediately know where the mismatch occurred.

<table><thead><tr><th width="224.228515625">RPC Call</th><th>Description</th></tr></thead><tbody><tr><td><a href="https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/indexing-data/getting-proofs/get_proof_for_blob"><code>get_proof_for_blob</code></a></td><td>Returns a <code>CompoundProof</code> bundle for one specific blob account.</td></tr><tr><td><a href="https://docs.termina.technology/documentation/network-extension-stack/modules/data-anchor/indexing-data/getting-proofs/get_proof"><code>get_proof</code></a></td><td>Returns a full <code>CompoundProof</code> for every blob in a slot—covering inclusion, bank/slot hashes, etc.</td></tr></tbody></table>

#### Accounts Delta Proof

This proof confirms which accounts changed during a block and which did not. The indexer reads the on-chain Merkle tree of account updates for the slot, collects the sibling hashes needed to rebuild the root, then compares that root to the `accounts_delta_hash` recorded on Solana.

If your user’s account or program state appears in that update, inclusion proof passes; if not, exclusion proof passes.

```rust
// Check account updates.

use data_anchor_proofs::accounts_delta_hash::{
    inclusion::InclusionProof, 
    exclusion::ExclusionProof,
};

inclusion_proof.verify(expected_accounts_delta_hash);
exclusion_proof.verify(expected_accounts_delta_hash)?;
```

#### Bank Hash Proof

Every block on Solana has a `bankhash` that summarizes its state transitions.

The proof provides the parent block’s bank hash, the accounts delta hash from above, the block’s signature count, and the blockhash. If those four values hash together, a match means the block’s entire state—including your blobs, was recorded correctly.

```rust
// Recreate and verify the bank hash.

use data_anchor_proofs::bank_hash::BankHashProof;

bank_hash_proof.verify(expected_bank_hash);
```

#### Slot Hash Proof

The Slot Hash Proof links that reconstructed bank hash back to the slot number. Solana writes every block’s bank hash into the `SlotHashes` sysvar.

This proof pulls that on-chain entry and verifies it matches your reconstructed hash, closing the loop between the local proof and the on-chain value.

```rust
// Confirm it was recorded in `SlotHashes`.

use data_anchor_proofs::slot_hash::SlotHashProof;

slot_hash_proof.verify(slot, bank_hash, accounts_delta_hash)?;
```

#### Blob Proof

When you send a data blob to the ledger space, it also logs a digest in each blob account. The Blob Proof takes the exact byte array you fetched, recomputes its digest, and confirms it matches the on-chain record.

This guarantees that what you received—whether JSON, image bytes, or telemetry—is exactly what was anchored in Solana.

```rust
// Ensure each blob’s raw bytes match.

use data_anchor_proofs::blob::BlobProof;

blob_proof.verify(&blob_bytes)?;
```

#### Compound Proof

Rather than juggling four separate checks, the Compound Proof bundles everything into one response object. If blobs exist at a slot, you receive an inclusion proof: accounts delta, bank hash, slot hash, and blob proofs all in one.

If no blobs were present, you receive a completeness proof, which shows that absence was legitimate.

```rust
// Bundle the above steps into one.

use data_anchor_proofs::compound::{
    inclusion::CompoundInclusionProof, 
    completeness::CompoundCompletenessProof, 
    inclusion::ProofBlob,
};

compound_proof.verify(
    blober_program, 
    blockhash, 
    &[ProofBlob { blob: blob_pubkey, data: Some(blob_bytes) }],
)?;
```
