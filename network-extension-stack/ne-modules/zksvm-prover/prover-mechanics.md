# Prover Mechanics

#### Technical Framework

To integrate ZKPs into Solana, Termina has the ability to generate proofs for any type of SVM transaction, such as account creation, SPL token transfers, or arbitrary program execution.

There were several issues that had to be resolved to integrate the SVM with a zkVM like SP1. At a high-level, these include:

1. **Randomness and Time**: Adjusting Solana’s use of time and randomness to remove nondeterminism.
2. **Threads and Files**: Adapting multithreaded processes and file-based operations to work within the constraints of the zkVM.
3. **Bit Depth**: Resolving memory access differences between the SVM’s eBPF interpreter’s 64-bit architecture and SP1’s 32-bit environment.

For additional information on the challenges above, please see this technical [blog](https://www.termina.technology/post/svm-on-sp1).

#### zkSVM Benchmarks

In terms of high-level benchmarks, 100 iterations of a 3-instruction SOL transaction take 2B cycles to prove while 100 iterations of an 8-instruction SPL transaction take 7B cycles. The average size of a Solana transaction on mainnet is around 4 instructions, which sits between the instruction counts that were tested in the two  benchmarks.

<figure><img src="../../../.gitbook/assets/Termina Diagrams (2).png" alt=""><figcaption></figcaption></figure>

| Metrics (Per Block) | SVM           | Base  | OP Mainnet |
| ------------------- | ------------- | ----- | ---------- |
| Transaction Count   | 100           | 100   | 16         |
| Cycle Count         | 3B            | 840M  | 280M       |
| Proving Time        | 130s (900s)   | 280s  | 90s        |
| Dollar Cost         | $0.95 ($0.80) | $0.64 | $0.21      |

On Succinct’s Prover Network, generating a core proof for a block of 100 complex SPL transactions only takes a few minutes and costs less than a dollar. You have the option to prioritize for proving time or dollar cost.

In addition, these values are rapidly changing as both prover hardware and software are making significant leaps in latency and cost.
