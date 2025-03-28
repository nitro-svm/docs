# Overview

**Network Extensions (NEs)** are infrastructure solutions that enhance Solana’s core capabilities. They can function **independently** or as **part of a rollup**, allowing developers to optimize performance, privacy, and data handling while keeping applications fully integrated with Solana L1.

<figure><img src="../.gitbook/assets/Termina Diagrams (6).png" alt=""><figcaption></figcaption></figure>

**NEs** are ideal for:

* **High-throughput applications** – Reduce congestion and improve execution speed.
* **Private or permissioned computation** – Run isolated workloads without modifying Solana’s state.
* **Data-intensive networks** – Store, compress, and retrieve large volumes of off-chain data efficiently.

#### **How It Works**

1. **Pick a module** – Use **execution, privacy, or data-focused** modules based on your needs.
2. **Deploy within Solana** – Plug-n-Play with modules which interact directly with Solana L1.
3. **Optimize performance** – Execute transactions efficiently while staying within Solana’s framework.

While the NE stack is a framework, **NE modules** are individual, specialized components. Developers can use them individually or together as needed.

For instance, the modules can work together to build a standard rollup but also function independently—whether to isolate execution in a permissioned environment, generate ZK proofs over SVM computation for privacy, or compress data for data-intensive applications.

The module system has a unique approach compared to existing architectures:

<figure><img src="../.gitbook/assets/Termina Diagrams (1) (1).png" alt=""><figcaption></figcaption></figure>
