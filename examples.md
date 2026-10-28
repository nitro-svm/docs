# Examples

Complete Rust programs built on `simulator-client` 0.29, one per use case. Each card opens the example's README, with its method, flags and a command to run.

<table data-view="cards"><thead><tr><th></th><th></th><th data-hidden data-card-target data-type="content-ref"></th></tr></thead><tbody><tr><td><strong>Hello Session</strong></td><td>The smallest complete program: pick a slot range, start a session, read an account through the session's RPC endpoint, and replay to the end.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/hello_session">hello_session</a></td></tr><tr><td><strong>Counterfactual Flow</strong></td><td>Shift a venue's price, reroute every historical swap via an aggregator, and count the flow the venue wins or loses against a baseline.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_flow">counterfactual_flow</a></td></tr><tr><td><strong>Counterfactual Capital</strong></td><td>Scale a venue's inventory and its quoting curve with account overrides, reroute every swap on the pair directly to the venue, and see which one limits its fills.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_capital">counterfactual_capital</a></td></tr><tr><td><strong>Venue Templates</strong></td><td>Reroute each original swap directly to a venue with the bundled venue templates or your own file, and score each template against the original fill.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/reroute_venue_templates">reroute_venue_templates</a></td></tr><tr><td><strong>Measuring Spread and Depth</strong></td><td>Quote one venue at a range of sizes at every slot, and report its spread and depth curve.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/amm_liquidity">amm_liquidity</a></td></tr><tr><td><strong>Datasets</strong></td><td>Get dataset credentials and the S3 path of each dataset, then request a custom depth dataset and poll the job until it is published.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/datasets">datasets</a></td></tr><tr><td><strong>Comparing Quotes</strong></td><td>Pause before each Jupiter swap and price a Titan route against the same state, to compare what each router pays out and which venues it uses.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/quote_compare">quote_compare</a></td></tr><tr><td><strong>Measuring Quote Decay</strong></td><td>Replay each original swap against later slots (the next 50 by default), to measure how fast its output decays as landing is delayed.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/quote_decay">quote_decay</a></td></tr><tr><td><strong>Capture Regressions</strong></td><td>Replay mainnet traffic against a new program binary, and report each transaction's result, logs and token balance changes.</td><td><a href="https://github.com/nitro-svm/examples/tree/main/src/bin/regression_test">regression_test</a></td></tr></tbody></table>

## Requirements

Every example reads your API key from `SIMULATOR_API_KEY`. Some also call services outside Termina.

| Example | Also calls | Related flow |
| --- | --- | --- |
| [`hello_session`](https://github.com/nitro-svm/examples/tree/main/src/bin/hello_session) | — | [Quickstart](overview/quickstart.md) |
| [`counterfactual_flow`](https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_flow) | Jupiter label API, with `--program-id` | [Rerouting via an Aggregator](flows/rerouting-historical-flow/rerouting-via-an-aggregator.md) |
| [`counterfactual_capital`](https://github.com/nitro-svm/examples/tree/main/src/bin/counterfactual_capital) | — | [Rerouting Directly to a Venue](flows/rerouting-historical-flow/rerouting-directly-to-a-venue.md) |
| [`reroute_venue_templates`](https://github.com/nitro-svm/examples/tree/main/src/bin/reroute_venue_templates) | — | [Rerouting Directly to a Venue](flows/rerouting-historical-flow/rerouting-directly-to-a-venue.md) |
| [`amm_liquidity`](https://github.com/nitro-svm/examples/tree/main/src/bin/amm_liquidity) | Solana mainnet RPC, Binance | [Generating Datasets](flows/generating-datasets/README.md) |
| [`datasets`](https://github.com/nitro-svm/examples/tree/main/src/bin/datasets) | AWS CLI or any S3 client, to download | [Generating Datasets](flows/generating-datasets/README.md) |
| [`quote_compare`](https://github.com/nitro-svm/examples/tree/main/src/bin/quote_compare) | Solana mainnet RPC | [Replaying the Market](flows/replaying-the-market.md) |
| [`quote_decay`](https://github.com/nitro-svm/examples/tree/main/src/bin/quote_decay) | Solana mainnet RPC | [Replaying the Market](flows/replaying-the-market.md) |
| [`regression_test`](https://github.com/nitro-svm/examples/tree/main/src/bin/regression_test) | — | [Replaying the Market](flows/replaying-the-market.md) |

"Solana mainnet RPC" is the public endpoint `api.mainnet-beta.solana.com`, which rate-limits heavy use. `datasets measure` also needs the depth feature enabled on your API key.

## Run an Example

```bash
git clone https://github.com/nitro-svm/examples
cd examples
export SIMULATOR_API_KEY=<API_KEY>
cargo run --bin hello_session
```

`hello_session` picks its own slot range. Most other examples take `--start-slot`, which must be the first slot of an available range. Print the latest one with:

```bash
curl -s https://simulator.termina.technology/available-ranges | jq 'map(.bundle_start_slot) | max'
```

`counterfactual_flow` and `counterfactual_capital` need a range with account state; their READMEs have the exact command. Run `cargo run --bin <name> -- --help` for an example's flags.
