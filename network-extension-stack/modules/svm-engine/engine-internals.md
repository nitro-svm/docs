# Engine Internals

<figure><img src="../../../.gitbook/assets/Termina Diagrams (3).png" alt=""><figcaption></figcaption></figure>

The **SVM Engine** runs a pure Solana transaction processing environment without relying on an external network or settlement layer. Whether optimizing DeFi protocols or supporting enterprise-grade applications, it aligns performance with Solana’s vision of a unified global state.

### How It Works

* **Native Integration:** Directly interacts with **Solana’s validator set** and account state and avoid the overhead of sequencers and fault proof mechanisms.
* **Unified Liquidity:** Keeps **assets anchored to Solana L1** instead of spreading liquidity across multiple networks.
* **Allocates dedicated resources** to prevent risks of congestion and ensure predictable performance.
