# Installation

### CLI Client

`sim` is a command-line client for managing simulation sessions without writing code. The tool allows replaying historical slots, injecting modified programs, and analyzing transaction outcomes.

Pre-built binaries are available for Linux and macOS (Apple Silicon).

```bash
curl -fsSL https://cli.simulator.termina.technology/install.sh | bash
```

If it's already installed, update the `sim` binary with the latest published version.

```bash
sudo sim update
```

All commands that connect to the simulator require an API key. Pass it via the `--api-key` flag or the `SIMULATOR_API_KEY` environment variable.

```bash
export SIMULATOR_API_KEY=<API_KEY>
```

### Rust Client

The `simulator-client` and `simulator-api` crates provide a native Rust interface to the Termina simulator, for teams who want to integrate backtesting directly into their Rust code rather than using the `sim` CLI.

* **`simulator-client`** is the high-level async client. It wraps the WebSocket protocol with ergonomic builders for common workflows: creating sessions, advancing slots, injecting transactions, and reading account state.
* **`simulator-api`** defines the raw protocol types (request/response structs, error variants, session parameters). Use it if you need direct access to the wire format or want to implement your own client.

For a complete set of end-to-end examples, see the starter code [repository](https://github.com/nitro-svm/examples).

```toml
[dependencies]
simulator-client = "0.15"
simulator-api = "0.15" # If you only need the protocol types (e.g. to build a custom client):
```
