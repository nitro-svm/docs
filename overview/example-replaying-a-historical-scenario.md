# Example: Replaying a Historical Scenario

The workflow starts by selecting a historical slot range and replaying it exactly as it executed on mainnet. This allows teams to observe system behavior under real conditions such as volatility, congestion, and state contention.

The same time window can then be replayed multiple times with modified parameters or logic. Each run produces structured outputs that can be compared across configurations, for example execution outcomes, PnL, or protocol-level metrics.

This replay-based approach can be applied to a wide range of cases, including:

* Investigating unexpected behavior during historical events
* Stress-testing protocol parameters across volatile periods
* Evaluating changes to execution or decision logic before deploying to mainnet
* Comparing alternative configurations using identical market conditions

The goal is to enable controlled experimentation on real historical data, so teams can reason about changes and their consequences without relying on trial-and-error in production.
