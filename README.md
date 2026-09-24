# Introduction

### **Background**

Backtests can be misleading. Most systems rely on mathematical models that assume fixed latency, frictionless fills, and predictable execution. But Solana doesn’t behave that way. Real outcomes depend on slot timing, account locking, compute limits, and network congestion: variables that are difficult to capture with math alone.

The gap between simulation and execution creates false confidence during strategy validation:

* Math models don’t reflect the nuanced reality of onchain protocols
* Transaction cost and market impact are underestimated, especially for large or high-frequency trades
* Iteration cycles are slow and paper trading takes days or weeks to reveal flaws
* Constant retuning is required to accommodate the volatility of crypto and DeFi

As a result, strategies that look profitable in backtests often underperform once deployed on mainnet.

### Summary

Termina’s simulation engine re-executes historical Solana flow to provide a slot-accurate state archive + deterministic execution layer for high-fidelity sims.

This enables:

* Quantify execution outcomes such as slippage, latency effects, and PnL distributions across different network conditions
* Validate strategy behavior against long historical periods without waiting for live market cycles
* Replay volatile periods or inject transactions to observe system behavior under stress

### Mental Model

* Pick a historical window
* Replay it deterministically
* Change reality
* Observe the outcome
* Repeat

### Get Connected

* [Schedule](https://calendly.com/rustem-awkb/30min) a demo
* [Request](https://t.me/rustemzzzz) an API key to integrate and run simulations

Our team is quick to respond and should be able to answer questions within a few hours.
