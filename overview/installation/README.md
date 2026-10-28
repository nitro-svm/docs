# Installation

Every path needs an API key. [Request](https://t.me/rustemzzzz) one, then install the client you want:

* [CLI](#cli): `sim`, for runs from a shell or CI. No code.
* [Rust Client](#rust-client): the `simulator-client` crate.
* [Raw API](#raw-api): JSON over a WebSocket and HTTP, from any language. Nothing to install.

Then follow the [Quickstart](../quickstart.md).

### CLI

`sim` runs sessions from the command line: replay historical slots, swap in a new program binary or account overrides, and compare the results.

Prebuilt binaries are available for Linux (x86_64) and macOS (Apple Silicon). The install script puts `sim` in `INSTALL_DIR`, which defaults to the current directory, so point it at a directory on your `PATH`:

```bash
mkdir -p "$HOME/.local/bin"
curl -fsSL https://cli.simulator.termina.technology/install.sh | INSTALL_DIR="$HOME/.local/bin" bash
sim --version
```

Update to the latest prebuilt binary:

```bash
sim update
```

It replaces the binary in place, so use `sudo sim update` only if `sim` lives in a system directory.

Prebuilt binaries (the install script and `sim update`) track the main branch, so they can be ahead of the latest release. For a reproducible install, such as in CI, pin a release with Cargo (Rust 1.97 or newer):

```bash
cargo install sim-cli --locked --version 0.22.0
```

Set your API key, and check it with `sim usage`. You can also pass it with `--api-key`.

```bash
export SIMULATOR_API_KEY=<API_KEY>
sim usage
```

A rejected key prints an `HTTP 401` error and exits with status 1. See [Troubleshooting](../../troubleshooting.md) for this and other errors.

#### In CI

A GitHub Actions job that installs a pinned `sim` without `sudo` and runs a fidelity check: it fails when a replayed transaction differs from its onchain outcome.

```yaml
- uses: dtolnay/rust-toolchain@stable
- run: cargo install sim-cli --locked --version 0.22.0
- run: sim run --start-slot <START_SLOT> --end-slot <END_SLOT> --program-id <PROGRAM_ID> --fail-on-diff
  env:
    SIMULATOR_API_KEY: ${{ secrets.SIMULATOR_API_KEY }}
```

`sim` exits with status 1 on an error or a difference. To gate a new program binary instead, compare it with a baseline as in [Use in CI](../../cli/README.md#use-in-ci). See the [CLI](../../cli/README.md) page for every exit status and environment variable.

### Rust Client

* **`simulator-client`** is the async Rust client. It creates sessions, steps them, injects transactions and reads account state, and reconnects on its own.
* **`simulator-api`** holds the protocol types. `simulator-client` re-exports it as `simulator_client::simulator_api`, so add it yourself only to build your own client.

Both crates need Rust 1.97 or newer.

```toml
[dependencies]
simulator-client = "0.29"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
rustls = { version = "0.23", default-features = false, features = ["ring"] }
```

Install a rustls crypto provider at the top of `main`, before you connect. Without it, the first connection can panic.

```rust
let _ = rustls::crypto::ring::default_provider().install_default();
```

If you also depend on Solana crates, use the versions in the [Rust Reference](rust-reference.md#install), so their types match the client's. Upgrading from an older version? See [Migration](../../migration.md).

Runnable programs for each use case are in the starter code [repository](https://github.com/nitro-svm/examples).

### Raw API

Clients in other languages talk to the server directly:

* The session control channel: a WebSocket at `wss://simulator.termina.technology/backtest`.
* The session's RPC endpoint: standard Solana JSON-RPC over HTTP, and subscriptions over WebSocket, one per session.
* HTTP endpoints for slot ranges, bundles, datasets and usage, at `https://simulator.termina.technology`.

Send your key in the `X-API-Key` header, on the WebSocket and on every HTTP request. Start with the [Raw WebSocket Walkthrough](api-reference/raw-websocket-walkthrough.md), then the [API Reference](api-reference/README.md).
