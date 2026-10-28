# Introduction

### Background

In TradFi, a trading strategy gets tested before it touches capital. On Solana DeFi, most desks test in production because historical state is hard to reproduce. A market maker who suspects their spreads are too tight has to widen them live, wait days, and compare PnL, while volume, competitors, and volatility shift underneath. The result is slow and costly, and the outcome is hard to attribute: any change in PnL could come from the spread or the market.

Current backtest methods try to approximate this via price feeds and transaction history, but testing "what would've happened?" requires the exact state of the chain at a past slot and a way to run transactions against it. Solana doesn't provide either natively.

### Use Cases

Termina replays historical Solana slots and can pause the chain at any point within a slot. From there, a transaction can be simulated against that state, with changed balances, parameters, or program code. Offchain aggregators are part of the replay, so routing and quoting also behave the way they did in production.

Questions that used to be guesswork can then be measured:

* How much flow would a wider spread or a new curve have won or lost last week? ([`counterfactual_flow`](https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_flow))
* What would the venue have quoted at 10x the size, and where does depth run out? ([`amm_liquidity`](https://github.com/nitro-svm/examples/tree/main/src/bin/amm_liquidity))
* Does a new program version break real mainnet traffic before it goes live? ([`regression_test`](https://github.com/nitro-svm/examples/tree/main/src/bin/regression_test))
* Would a router have won a given trade against competing routes? ([`quote_compare`](https://github.com/nitro-svm/examples/tree/main/src/bin/quote_compare))
* How fast does a quote decay as latency grows between quote and landing? ([`quote_decay`](https://github.com/nitro-svm/examples/tree/main/src/bin/quote_decay))

Each link is a working example. The [Examples](examples.md) page lists them all, with what each one needs.

### How It Works

* A **session** replays a slot range of Solana mainnet on the server. You step it forward block by block, and it pauses between steps.
* While it's paused, read and change its state through **the session's RPC endpoint**: standard Solana JSON-RPC. Send user transactions, apply account overrides, or swap in a new program binary.
* The server replays **bundles** of historical blocks: prebuilt bundles cover the past 30 days, and you can build custom bundles for other ranges.
* With `sim`, one **run** drives one or more sessions over a range and writes the results to a file. Compare a baseline run with an experiment run to see what your change did.

### Get Started

1. [Install](overview/installation/README.md) the CLI or the Rust client.
2. Run your first session with the [Quickstart](overview/quickstart.md).
3. Pick a flow: [Replaying the Market](flows/replaying-the-market.md), [Rerouting Historical Flow](flows/rerouting-historical-flow/README.md) or [Generating Datasets](flows/generating-datasets/README.md).

### Get Connected

* [Schedule](https://calendly.com/rustem-awkb/30min) a demo
* [Request](https://t.me/rustemzzzz) an API key to integrate and run simulations

Our team is quick to respond and should be able to answer questions within a few hours.
