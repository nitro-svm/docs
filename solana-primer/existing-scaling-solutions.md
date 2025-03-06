# Existing Scaling Solutions

While Solana’s core infrastructure provides high speed and composability, the growing complexity and diversity of SVM dApps requires tailored solutions to provide congestion resistance, enhanced privacy, and advanced data processing.

One common misconception is that Solana doesn’t need L2s when in fact the ecosystem already has a number of notable L2 projects. Please see [this blog](https://www.termina.technology/post/svm-usecases) for a high-level exploration of these dApps. The scaling solutions that have emerged can be organized into three main categories: rollups, batchers, and Solana Permissioned Environments (SPEs). However, each of these present a unique set of tradeoffs and often diverge from Solana’s vision of a **globally unified state machine** and its commitment to preserving liquidity and interoperability.

<figure><img src="../.gitbook/assets/34r.png" alt=""><figcaption></figcaption></figure>

### Rollups

A rollup executes transactions off-chain but submits state summaries and fault proofs to Solana for verification and settlement. This allows the rollup’s state to be challenged to prevent fraudulent behavior from the sequencer. Rollups are used by projects like Zeta’s Bullet L2 to run its perpetuals DEX and leverage Solana as a data availability and settlement layer. Despite their advantages, rollups can fragment state as they create parallel execution environments.

### **Batchers**

Similar to a rollup, a batcher consolidates transactions off-chain. However, unlike a rollup, a batcher does not submit the resulting summary to Solana—instead, it executes the net movements on there directly. Although execution occurs after a delay, the final state on L1 is eventually consistent with that of the batcher. This design anchors the source of truth and security to Solana itself.

Projects like Code, the P2P payments network, and Cube, the hybrid CEX/DEX, use a batcher to aggregate transactions off-chain then execute them on-chain so that a user's funds are always on the L1.

### **Appchains and SPEs**

An appchain or Solana Permissioned Environment (SPE) is an independent network that runs Solana code but operates with its own security and trust assumptions. As the name suggests, SPEs have a restricted validator set, which provides a controlled environment.

Pyth, the oracle network, along with several tradFi and RWA projects all utilize SPEs to power their applications. While suitable for specific use cases, SPEs and appchains operate outside Solana’s unified state, which can be an advantage or a limitation depending on the desired design.
