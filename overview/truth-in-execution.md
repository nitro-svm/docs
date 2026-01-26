# Truth in Execution

### Motivation

Termina’s simulation engine re-executes historical Solana blocks with slot-level accuracy, allowing teams to evaluate strategies under real execution conditions. It provides a slot-accurate state archive + deterministic execution layer for high-fidelity sims.

What this enables:

* Quantify execution outcomes such as slippage, latency effects, and PnL distributions across different network conditions
* Validate strategy behavior against long historical periods without waiting for live market cycles
* Replay volatile periods or inject transactions to observe system behavior under stress

Teams bring their own strategy and use the results to assess whether behavior in simulation aligns with onchain execution.

### Mental Model

* Pick a historical window
* Replay it deterministically
* Change reality
* Observe the outcome
* Repeat
