# zkSVM Prover

Termina’s zkSVM module is designed to bring Solana a new level of scalability and privacy by leveraging [**zero-knowledge proofs**](#user-content-fn-1)[^1] **(ZKPs)** for arbitrary SVM transactions.

#### **TL;DR:**

* Leverage privacy-preserving ZKP systems for sensitive or complex transaction workflows.
* Scale applications within Solana’s infrastructure while relying on the L1 as the source of truth.
* Achieve faster dispute resolution in rollups with non-interactive ZKPs (from days → hours).

Every transaction must be verified for security, which means that each Solana validator has to rerun the transaction. For complex transactions, this can add up and degrade performance. The zkSVM resolves this issue by allowing Solana validators to check a small, cheap proof instead of execute large, expensive transactions. The zkSVM prover **processes transactions,** generates a [**proof of correctness**](#user-content-fn-2)[^2]**,** and **submits that proof** to Solana. The blockchain doesn’t need to run the transactions and only needs to verify the proof.

This allows teams to build privacy-first applications with advanced ZKPs (such as Groth16) that don't leak any information. It also enables [**non-interactive proofs**](#user-content-fn-3)[^3], which means rollup disputes that normally take days to resolve can now be settled in hours.

Instead of a handrolled [**circuit**](#user-content-fn-4)[^4], the zkSVM runs on a general-purpose zkVM—which makes it easy to apply bug fixes or new features to the SVM as they become available upstream. It was initially built on [**Succinct’s SP1**](https://docs.succinct.xyz/docs/sp1/introduction) but will work with any zkVM following the **RISCV architecture.**

<figure><img src="../../../.gitbook/assets/5 (1) (1).png" alt=""><figcaption></figcaption></figure>

[^1]: A cryptographic method that proves a statement without revealing the underlying data.

[^2]: A cryptographic guarantee that an off-chain transaction followed the correct rules before being accepted on-chain.

[^3]: A proof that doesn’t require back-and-forth interaction between prover and verifier, reducing settlement times.

[^4]: Mathematical structures in zero-knowledge systems that encode and verify computations without exposing private data.
