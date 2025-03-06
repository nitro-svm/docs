# Rollups

While Termina is primarily focused on Network Extensions (NEs) as a new scaling approach for Solana, we recognize that some use cases still require rollup-based solutions. SVM rollups execute business logic off-chain but submit transaction batches and the resulting state to Solana, which serves as the data availability and settlement layer. This provides a dedicated execution environment for developers and allows the rollup’s state to be challenged to prevent fraudulent behavior, but connecting state and liquidity between Solana L1 and the rollup requires more complexity.

As an example, projects like Grass and Zeta have built their own rollups to address application-specific needs, which show that the benefits of a full fledged rollup can outweigh the tradeoffs in certain scenarios. For teams that prefer a rollup architecture over NEs, Termina provides a **full rollup stack** that simplifies deployment. It already integrates all three NE modules—**SVM Engine, zkSVM Prover, and Data Module**—so developers don't need to perform any additional work to manually integrate them.

Please [reach out](https://calendly.com/rustem-awkb/30min) to our team if you are interested in deploying a rollup.
