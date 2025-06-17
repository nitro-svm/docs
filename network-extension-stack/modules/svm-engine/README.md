# SVM Engine

The SVM Engine runs a dedicated execution environment for faster and more predictable transaction processing.

While Solana’s architecture enables high throughput and parallel execution, certain applications like [**HFT** (high-frequency trading)](#user-content-fn-1)[^1] and **data-heavy platforms** require guaranteed low latency and consistent finality. These requirements can be difficult to meet during peak demand in a global execution environment.

Traditionally, rollups addressed these challenges by moving computation off-chain and submitting proofs back to the L1, but this introduced additional complexity, delayed settlement, and fragmented liquidity. The **SVM Engine module** takes a different approach by enabling developers to create isolated **execution environments** that remain composable within Solana and its liquidity, while providing greater control over resource allocation.

This is particularly valuable for **HFT**, **DeFi**, and **permissioned environments**, where custom execution logic and resource allocation are critical for scalability. The SVM Engine module allows developers to achieve rollup-like performance benefits without requiring off-chain proofs, separate settlement layers, or fragmented ecosystems.

[^1]: A trading method that rapidly processes orders to capitalize on small price changes.
