# Session RPC and Subscriptions

Each session has its own RPC endpoint, named by `rpc_endpoint` in `sessionCreated`. It serves Solana JSON-RPC against the session's current state, and Solana-style subscriptions over WebSocket.

* RPC: `POST https://simulator.termina.technology/backtest/<session_id>`, a JSON-RPC 2.0 body.
* Subscriptions: a WebSocket to `wss://simulator.termina.technology/backtest/<session_id>`.
* No API key. The session ID is the credential.
* Use it once the session sends `readyForContinue`. Reads see the state after the last executed block, or up to the pause point after `continueTo`. Before the first `continue`, that is the state at the end of the slot before `startSlot`. It serves until `completed`, then answers HTTP 502.

```bash
curl -s https://simulator.termina.technology/backtest/<session_id> \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"getAccountInfo","params":["<address>",{"encoding":"base64"}]}'
```

## RPC Methods

Methods take Solana's parameters and return Solana's shapes, except where the notes say otherwise.

### Accounts

| Method | Notes |
| --- | --- |
| `getAccountInfo` | Default encoding is `base64+zstd`. Pass `{"encoding": "base64"}` for plain base64. |
| `getMultipleAccounts` | Default encoding is `base64`. |
| `getBalance` | |
| `getProgramAccounts` | Default encoding is `base64+zstd`. Supports `filters`, `dataSlice` and `withContext`. |
| `getAddressLookupTable` | Same as `getAccountInfo` on the table's address. |
| `getTokenAccountBalance`, `getTokenAccountsByOwner`, `getTokenAccountsByDelegate`, `getTokenLargestAccounts`, `getTokenSupply` | |
| `getMinimumBalanceForRentExemption` | |

### Transactions

| Method | Notes |
| --- | --- |
| `sendTransaction` | Executes the transaction at once, in the current slot, after the block's own transactions. Signatures are not verified, so a transaction can act as any signer. Default encoding is `base64`. Returns the signature. |
| `simulateTransaction` | Runs the transaction without committing. Signatures are not verified; `sigVerify` and `replaceRecentBlockhash` are accepted and ignored. Default encoding is `base64`. Honors `innerInstructions` and `accounts`, and takes [`modifyAccountStates`](#simulatetransaction-with-account-overrides). |
| `getTransaction` | Finds historical and user transactions the session executed. |
| `getSignatureStatuses` | |
| `getFeeForMessage` | Number of required signatures times the lamports per signature. Legacy messages only. |
| `getTransactionCount` | Transactions the session executed. |

`sendTransaction`, `getTransaction` and `getSignatureStatuses` are rejected in a session created with `replayAccountState`.

User transactions run in this order within a slot:

1. The block's historical transactions.
2. User transactions sent with `sendTransaction`.
3. User transactions in the next `continue`.

### Chain State

| Method | Notes |
| --- | --- |
| `getSlot`, `getBlockHeight` | The session's current slot. Block height equals the slot. |
| `getLatestBlockhash` | The current slot's blockhash. |
| `getRecentBlockhash`, `getFees` | Deprecated Solana forms of `getLatestBlockhash`. |
| `getBlock` | The current block from the session; any other slot from the server's history. |
| `getBlocks`, `getBlocksWithLimit` | Only the current block's slot, when it is in the range. |
| `getBlockTime` | The current block's time, whatever slot you ask for. |
| `getFirstAvailableBlock`, `minimumLedgerSlot` | The current slot. |
| `getEpochInfo` | |
| `getVersion`, `getHealth` | |

### Account Overrides

`modifyAccounts` is a Termina method. It writes account overrides into the session at once, and returns how many accounts it wrote.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "modifyAccounts",
  "params": [{ "<address>": { "data": { "data": "<base64>", "encoding": "base64" }, "executable": false, "lamports": 1000000, "owner": "<owner>", "space": 80 } }]
}
```

### Fixed Responses

These exist so standard Solana clients work, but return fixed or empty values rather than the replayed chain's:

| Method | Returns |
| --- | --- |
| `isBlockhashValid` | Always `true`. |
| `getRecentPrioritizationFees` | One entry: the current slot, fee 0. |
| `getGenesisHash` | Mainnet's genesis hash. |
| `getEpochSchedule` | Solana's default schedule. |
| `getStakeMinimumDelegation` | 1 SOL. |
| `getHighestSnapshotSlot` | `{"full": <current slot>, "incremental": null}`. |
| `getMaxRetransmitSlot`, `getMaxShredInsertSlot` | The current slot. |
| `getIdentity`, `getSlotLeader`, `getSlotLeaders` | The all-zeros address. |
| `getSignaturesForAddress`, `getClusterNodes`, `getLargestAccounts`, `getRecentPerformanceSamples` | Empty lists. |
| `getInflationReward`, `getLeaderSchedule` | `null` entries. |
| `getBlockCommitment`, `getBlockProduction`, `getInflationGovernor`, `getInflationRate`, `getSupply`, `getVoteAccounts` | Fixed placeholder values. |
| `requestAirdrop` | An error. Fund accounts with an account override instead. |

Any other method returns JSON-RPC error `-32601`.

### Errors

| Code | When |
| --- | --- |
| `-32602` | Invalid parameters, such as a malformed address or transaction. |
| `-32600` | The session can't serve the method, such as `sendTransaction` under `replayAccountState`. |
| `-32603` | The session or the execution failed. The message starts with `Session error:` or `Execution error:`. |

### simulateTransaction with Account Overrides

`simulateTransaction` takes Solana's config plus `modifyAccountStates`: account overrides that apply to this simulation only.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "simulateTransaction",
  "params": [
    "<base64 transaction>",
    {
      "encoding": "base64",
      "innerInstructions": true,
      "accounts": { "addresses": ["<address>"], "encoding": "base64" },
      "modifyAccountStates": {
        "<address>": { "data": { "data": "<base64>", "encoding": "base64" }, "executable": false, "lamports": 2039280, "owner": "TokenkegQfeZyiNwAJbNbGKPFXCWuMvBuz6xNVu6daA", "space": 165 }
      }
    }
  ]
}
```

An override here can't be executable or owned by a program loader.

## Subscriptions

Open a WebSocket to the session's RPC endpoint and send JSON-RPC subscribe requests. Each subscribe returns a subscription ID; notifications follow as `{"jsonrpc": "2.0", "method": "<kind>Notification", "params": {"subscription": <id>, "result": …}}`. Only subscription methods work on this WebSocket: send RPC methods over HTTP.

| Subscribe | Params | Notification | `result` |
| --- | --- | --- | --- |
| `accountSubscribe` | `[address, config?]` | `accountNotification` | Solana's account notification. The current state arrives first. |
| `programSubscribe` | `[program_id, config?]` | `programNotification` | Solana's program notification. Every current account arrives first. |
| `signatureSubscribe` | `[signature, config?]` | `signatureNotification` | Solana's signature notification. One-shot. |
| `slotSubscribe` | `[]` | `slotNotification` | `{"slot", "parent", "root"}`. |
| `logsSubscribe` | `[filter, config?]` | `logsNotification` | Solana's logs notification. `filter` is `"all"`, `"allWithVotes"` or `{"mentions": [address]}`. |
| `transactionSubscribe` | `[filter, config?]` | `transactionNotification` | The full transaction with its metadata, as `getTransaction` returns it. `filter` as for `logsSubscribe`. |
| `accountDiffSubscribe` | `[address, {"address_type": "account" \| "program"}?]` | `accountDiffNotification` | One per write: `{"context", "account", "signature", "tx_index"?, "block_time"?, "pre", "post"}`. `pre` is `null` for a new account, `post` for a deleted one. With `"program"`, every account the program owns. |
| `actionSubscribe` | `[null, config?]` | `actionNotification` | One scheduled action's result. See below. |
| `replacementSubscribe` | `[null, config?]` | `replacementNotification` | One reroute result. See below. |
| `diagnosticSubscribe` | `[null, config?]` | `diagnosticNotification` | One diagnostic probe sample: `{"slot", "probeIndex", "label"?, "market", "usdValue", "routerSlot"?, "results"?, "accountHashes", "err"?}`. |

Every `result` except those of `slotNotification` and `transactionNotification` carries `context: {"slot"}`, the slot it was produced at.

Unsubscribe with the matching `*Unsubscribe` method and the subscription ID; it returns `true` if the subscription existed. `{"jsonrpc": "2.0", "id": 1, "method": "ping"}` returns `"pong"`.

When the session ends, each subscription gets a final `subscriptionComplete` notification with a `null` result, then the server closes the WebSocket.

### Subscription Options

The `config` object takes these options next to the method's own fields:

| Option | Meaning |
| --- | --- |
| `replayFromSlot` | A new subscription first replays the session's matching events, then live ones. Without this option it replays every one since the session started; with it, only those from this slot (inclusive). Use it after a reconnect, with the last slot you processed. `signatureSubscribe` never replays. |
| `intraSlotUpdates` | `accountSubscribe` and `programSubscribe` only. `true` sends one notification per write; the default sends one per account per slot, with the slot's final state. |
| `compression` | `"zstd"` sends notifications as binary frames of one zstd stream that spans the connection. Each frame decodes on its own, but only with the previous frames' context. Leave it unset unless your zstd library supports streaming decompression. |

### actionNotification

```json
{
  "context": { "slot": 380000010 },
  "slot": 380000010,
  "batchIndex": 2,
  "txIndex": 41,
  "actionIndex": 1,
  "label": "probe",
  "committed": false,
  "transactionOutcomes": [
    { "logs": ["Program … invoke [1]"], "unitsConsumed": 41200, "fee": 5000, "returnData": { "programId": "<program_id>", "data": "<base64>" } }
  ],
  "accounts": [{ "lamports": 1000000, "data": ["<base64>", "base64"], "owner": "<owner>", "executable": false, "rentEpoch": 0, "space": 80 }],
  "matched": { "data": "<base64>", "encoding": "base64" }
}
```

* `actionIndex` is the action's position in `actions`.
* `transactionOutcomes` has one entry per transaction, in order. `err` is absent on success.
* `accounts` follows `returnAccounts`; `null` for an account that doesn't exist.
* `batchIndex`, `txIndex` and `matched` are absent for slot anchors. `matched` is the transaction that matched the filter.

### replacementNotification

One result of rerouting an original swap. `kind` says which:

| `kind` | Meaning |
| --- | --- |
| `requote` | Rerouted via an aggregator: the swap re-quoted through the router and simulated. |
| `directFill` | Rerouted directly to a venue: one hop re-priced through a venue template. |
| `original` | The original swap replayed as the baseline. |

Every kind carries `slot`, `batchIndex`, `originalSignature`, `originalFailed`, `outcome` (`filled`, `reverted`, `unattributed` or `unsimulatable`) and, when it failed, `category` and `err`. `l1Fill` is what the original swap filled, and `changeVsL1Bps` how the result compares with it in basis points; positive means it beat the original swap. The full field list per kind is in [`simulator_api::replacement`](https://docs.rs/simulator-api/latest/simulator_api/replacement/index.html).

## Yellowstone gRPC

Each session also serves the Yellowstone `geyser.Geyser` gRPC service at `https://simulator.termina.technology`. Name the session in one of two ways:

* Prefix the method path: `/backtest/<session_id>/geyser.Geyser/Subscribe`.
* Or send the `x-backtest-session-id: <session_id>` metadata header with the plain path.

If the path and the header name different sessions, the request is rejected.
