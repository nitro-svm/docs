# FAQs

Network Extensions (NEs) expand Solana’s execution capabilities while staying integrated with its base layer. Rollups are a subset of NEs that process transactions off-chain and post proofs back to Solana.

While all rollups are NEs, not all NEs are rollups, some function as standalone modules that scale specific parts of an application.



<table><thead><tr><th width="184">FAQ</th><th>Network Extension (NE)</th><th>Rollup</th></tr></thead><tbody><tr><td>How does sequencing work?</td><td><strong>Sequencers are not required for NEs.</strong> Transactions can still be ordered, but execution can happen without a strict sequencing model.</td><td><strong>Rollups use sequencers to order transactions in a block</strong> before submitting them to Solana. Sequencing is mandatory and can be centralized or permissionless (with staking for security).</td></tr><tr><td>How does block building happen?</td><td><strong>NEs don’t create blocks in the traditional sense.</strong> Instead, they execute transactions off-chain and submit final results (state updates or proofs) to Solana.</td><td><strong>Rollups batch transactions into blocks</strong>, compute the state changes, and submit the block state root to Solana.</td></tr><tr><td>How does L1 ↔ NE message passing work?</td><td><strong>NEs don’t need bridges for Solana-native interactions.</strong> Instead, they use direct state reading/writing (e.g., SVM Engine can pull Solana state and update it later).</td><td>Rollups use bridges (e.g., Hyperlane) to pass messages between Solana L1 and the rollup.</td></tr><tr><td>How does existing infra (stables, RPCs, and others) connect with NEs?</td><td><strong>NEs stay on Solana, no bridges are required.</strong> Stablecoins and tokens remain native, and RPCs can query both L1 and NE execution layers.</td><td><strong>Rollups are required to bridge stablecoins and assets</strong> since they operate in a separate execution environment.</td></tr></tbody></table>

TL;**DR**:

* Rollups require sequencers, bridges, and block-building logic.
* Termina’s NEs are modular and don’t need sequencers or bridges, reducing complexity.
* NEs interact directly with Solana L1, keeping liquidity and composability intact.
