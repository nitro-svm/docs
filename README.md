# Introduction

### Use Cases

In TradFi, a trading strategy gets tested before it touches capital. On Solana DeFi, most desks test in production because historical state is hard to reproduce. A market maker who suspects their spreads are too tight has to widen them live, wait days, and compare PnL, while volume, competitors, and volatility shift underneath. The result is slow and costly, and the outcome is hard to attribute: any change in PnL could come from the spread or the market.

Current backtest methods try to approximate this via price feeds and transaction history, but testing "what would've happened?" requires the exact state of the chain at a past slot and a way to run transactions against it. Solana doesn't provide either natively.

Termina replays historical Solana slots and can pause the chain at any point within a slot. From there, a transaction can be simulated against that state, with changed balances, parameters, or program code. Offchain aggregators are part of the replay, so routing and quoting also behave the way they did in production.

Questions that used to be guesswork can then be measured:

* How much flow would a wider spread or a new curve have won or lost last week?
* What would the venue have quoted at 10x the size, and where does depth run out?
* Does a new program version break real mainnet traffic before it goes live?
* Would a router have won a given trade against competing routes?
* How fast does a quote decay as latency grows between quote and landing?

Each of these has a working example with [starter code](https://github.com/nitro-svm/examples).

### Get Connected

* [Schedule](https://calendly.com/rustem-awkb/30min) a demo
* [Request](https://t.me/rustemzzzz) an API key to integrate and run simulations

Our team is quick to respond and should be able to answer questions within a few hours.
