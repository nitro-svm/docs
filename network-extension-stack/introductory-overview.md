# Introductory Overview

<figure><img src="../.gitbook/assets/Termina Diagrams (6).png" alt=""><figcaption></figcaption></figure>

**Network Extensions (NEs)** are advanced infrastructure solutions that enhance Solana’s core capabilities with tools for scalability, privacy, and specialized processing.&#x20;

They can take different forms—standalone modules that scale specific sections of an application or full rollups that extend execution beyond the base layer.

They empower developers to create robust and customized dApps while maintaining Solana’s foundational strengths, including composability, security, and unified liquidity.

**NEs** are ideal for:

* Workloads requiring high throughput, low latency, and congestion resistance.
* Applications involving permissioned or private computation.
* Networks generating and processing a large volume of data.

While the NE stack is a framework enhancing Solana’s core capabilities, **NE modules** are the individual, specialized components. Each serves as a template and is designed to address specific needs such as performance, privacy, or data compression.

Developers have the flexibility to adopt the entire framework or selectively integrate each module to tailor the solution to their unique requirements.&#x20;

For instance, the modules can work together to build a standard rollup but also function independently—whether to isolate execution in a permissioned environment, generate ZK proofs over SVM computation for privacy, or compress data for data-intensive applications.

The module system has a unique approach compared to existing architectures:

* **Native Integration with Solana:** The NE stack eliminates the need for complex bridges or standalone networks, preserving the interoperability and liquidity of Solana L1.
* **Efficient Computation with SVM Engine:** The SVM Engine module allows developers to create isolated execution environments that scale efficiently to enable low latency and high-performance trading and DeFi applications. This allows for **parallel transaction execution**, optimizing throughput, and still retaining the security and composability with Solana.
* **Scalable Privacy with zkSVM Prover:** The zkSVM module enables developers to incorporate zero-knowledge proofs directly into Solana, allowing for faster fraud-proof resolution, enhanced scalability, and **privacy-preserving** transactions, without the tradeoffs associated with siloed chains.
* **Data Rollup for Data-Intensive Use Cases:** The data module enables large-scale data storage and retrieval while minimizing the on-chain footprint and cost. It’s especially useful for applications like DePIN or IoT that need to handle vast amounts of data in a cost-effective and secure manner.

<figure><img src="../.gitbook/assets/Termina Diagrams (4).png" alt=""><figcaption></figcaption></figure>

Termina empowers developers to choose what works best and build on their own terms—without leaving Solana.
