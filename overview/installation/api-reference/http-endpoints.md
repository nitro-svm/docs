# HTTP Endpoints

All paths are on `https://simulator.termina.technology`. Send your API key as the `X-API-Key` header. Bodies are JSON, with snake_case fields and enum values.

| Method | Path | What it does |
| --- | --- | --- |
| `GET` | [`/available-ranges`](#get-available-ranges) | The slot ranges you can replay. |
| `GET` | [`/snapshot-slot`](#get-snapshot-slot) | The latest slot the server has a snapshot for. |
| `POST` | [`/build-bundle`](#post-build-bundle) | Build a custom bundle. |
| `GET` | [`/build-bundle`](#get-build-bundle) | Your builds and bundles. |
| `GET` | [`/build-bundle/{request_id}`](#get-build-bundle-request_id) | One build's status. |
| `DELETE` | [`/bundles`](#delete-bundles) | Delete one of your bundles. |
| `POST` | [`/generate-depth`](#post-generate-depth) | Request a custom depth dataset. |
| `GET` | [`/generate-depth/coverage`](#get-generate-depth-coverage) | What a depth job samples. |
| `GET` | [`/generate-depth/{request_id}`](#get-generate-depth-request_id) | One depth job's status. |
| `POST` | [`/datasets/credentials`](#post-datasets-credentials) | Temporary read-only S3 credentials for datasets. |
| `GET` | [`/usage`](#get-usage) | Your usage, per UTC day. |
| `GET` | [`/usage/sessions`](#get-usage-sessions) | Every session that ended in a window. |
| `GET` | `/health` | Liveness. No API key. |

The session control channel (`/backtest`) and the session's RPC endpoint (`/backtest/<session_id>`) are on the same host; see [Session Control Channel](session-control-channel.md) and [Session RPC and Subscriptions](session-rpc.md).

## Errors

A failed request gets a non-2xx status with a plain-text body that says why.

| Status | Meaning |
| --- | --- |
| `400` | The request is invalid. The body says what to change. |
| `401` | The API key is missing or invalid. |
| `403` | Your API key doesn't have custom datasets enabled. Only the depth endpoints return it. |
| `404` | The job ID or bundle doesn't exist, or isn't yours. |
| `429` | Your key used up its request limit for bundle builds and depth jobs, which they share. Wait, then send the request again. |
| `503` | The feature isn't available on this server right now. |

## Ranges and Snapshots

### GET /available-ranges <a href="#get-available-ranges" id="get-available-ranges"></a>

```bash
curl -s -H "X-API-Key: $SIMULATOR_API_KEY" \
  "https://simulator.termina.technology/available-ranges?include_account_state=false"
```

The API key is optional here. Without it you get the prebuilt ranges only; with it, your custom bundles too. An invalid key gets `401`.

* `include_account_state` (default `false`): also list ranges that replay recorded account state.

```json
[
  {
    "bundle_start_slot": 380000000,
    "bundle_start_slot_utc": "2026-10-20T00:00:01Z",
    "max_bundle_end_slot": 380049999,
    "max_bundle_end_slot_utc": "2026-10-20T05:33:20Z",
    "max_bundle_size": 50000,
    "kind": "blocks",
    "scope": "global"
  }
]
```

* Ranges are sorted by `bundle_start_slot`, oldest first.
* A single session starts at `bundle_start_slot` and ends at or before `max_bundle_end_slot`.
* `max_bundle_end_slot` and `max_bundle_size` are `null` when the server reports no end for the range. `bundle_start_slot_utc` and `max_bundle_end_slot_utc` are `null` when the time is unknown.
* `kind`: `blocks` (replays transactions) or `account_state` (replays recorded account state).
* `scope`: `global` (prebuilt) or `user` (built for your API key).

### GET /snapshot-slot <a href="#get-snapshot-slot" id="get-snapshot-slot"></a>

Returns `{"snapshot_slot": 380512345}`: the latest slot the server has a snapshot for. `sim bundle build --latest` builds the slots right after it.

## Bundles

### POST /build-bundle <a href="#post-build-bundle" id="post-build-bundle"></a>

Builds a custom bundle. The build runs in the background; poll its status with the job ID.

```json
{ "start_slot": 380000000, "end_slot": 380049999, "bundle_size": 10000, "idempotency_key": "my-build-1", "kind": "blocks" }
```

| Field | Default | Meaning |
| --- | --- | --- |
| `start_slot`, `end_slot` | required | The range, inclusive. |
| `bundle_size` | the whole range | Slots per bundle. A session replays one bundle, so smaller bundles allow more parallel sessions. |
| `idempotency_key` | none | Resending the same key doesn't charge your quota again. |
| `kind` | `blocks` | `blocks`, or `account_state` to record account state for the exact range. |

The response is the build's status:

```json
{
  "request_id": "<job_id>",
  "start_slot": 380000000,
  "end_slot": 380049999,
  "bundle_size": 10000,
  "status": "in_progress",
  "created_at_unix_ms": 1761523200000,
  "kind": "blocks",
  "eta_secs": 1800
}
```

`status` is `in_progress`, `completed`, `failed` or `needs_investigation`. With `needs_investigation`, the outcome is unknown: assume neither success nor failure.

### GET /build-bundle <a href="#get-build-bundle" id="get-build-bundle"></a>

Lists your builds, newest first, and the bundles you own. Use it to recover a job ID you lost.

* `limit` (optional): the most builds to return.

```json
{
  "builds": [ { "request_id": "<job_id>", "status": "completed", "…": "…" } ],
  "bundles": [ { "start_slot": 380000000, "end_slot": 380009999, "kind": "blocks", "completed_at_unix_ms": 1761525000000, "size_bytes": 912345678 } ]
}
```

`bundles` is `null` when the listing is unavailable, and `[]` when you own none.

### GET /build-bundle/{request_id} <a href="#get-build-bundle-request_id" id="get-build-bundle-request_id"></a>

Returns one build's status, in the shape `POST /build-bundle` returns.

### DELETE /bundles <a href="#delete-bundles" id="delete-bundles"></a>

Deletes one of your bundles, named by its exact range and kind in the query: `?start_slot=380000000&end_slot=380009999&kind=blocks`. `kind` defaults to `blocks`. Returns `{"start_slot", "end_slot", "kind"}` of the deleted bundle. A bundle you don't own gets `404`.

## Depth Datasets

### POST /generate-depth <a href="#post-generate-depth" id="post-generate-depth"></a>

Requests a custom depth dataset for a slot range. The job runs in the background; poll its status with the job ID.

```json
{ "start_slot": 380000000, "end_slot": 380431999, "source": "onchain", "granularity": "slot", "markets": [] }
```

| Field | Default | Meaning |
| --- | --- | --- |
| `start_slot`, `end_slot` | required | The range, inclusive. The capture can extend past it to snapshot boundaries. |
| `source` | `onchain` | `onchain`: what each venue's onchain liquidity would fill. `router`: what the router would quote. |
| `granularity` | `slot` | `slot`, or `oracle` (one sample per oracle update; `onchain` only). |
| `markets` | every market | Market IDs to sample, as `GET /generate-depth/coverage` lists them. Another market gets `400`. |
| `program_overrides` | `[]` | Up to 8 `{"program_id": "<base58>", "elf": "<base64>"}`: program binaries to run in place of the deployed ones. `onchain` only. |
| `account_overrides` | `{}` | Up to 64 account overrides every probe sees. `onchain` only. |

A request spans at most 10,000,000 slots. Sending a request identical to an earlier one (same `source`, `granularity`, range, `markets` and overrides) returns the earlier job and its status, without running or charging it again. A failed job runs again, also without a new charge.

The response is the job's status:

```json
{
  "request_id": "<job_id>",
  "start_slot": 380000000,
  "end_slot": 380431999,
  "source": "onchain",
  "status": "in_progress",
  "capture": { "start": 379999000, "end": 380433000 },
  "modification_digest": "<digest>"
}
```

* `status` takes the same values as a bundle build.
* `capture` is the range actually captured.
* `modification_digest` is present when you passed overrides. It names the dataset's path, `modified/depth/onchain/slot/<digest>`.
* `granularity` is absent when it is `slot`.

### GET /generate-depth/coverage <a href="#get-generate-depth-coverage" id="get-generate-depth-coverage"></a>

What a depth job samples, per source:

```json
{
  "sources": [
    {
      "source": "onchain",
      "revision": 2,
      "markets": ["SOL-USDC"],
      "venues": ["<venue>"],
      "size_rungs": 12,
      "completed_ranges": [ { "start": 380000000, "end": 380431999 } ]
    }
  ]
}
```

`completed_ranges` lists the ranges you already published at this revision.

### GET /generate-depth/{request_id} <a href="#get-generate-depth-request_id" id="get-generate-depth-request_id"></a>

Returns one depth job's status, in the shape `POST /generate-depth` returns.

## Datasets

### POST /datasets/credentials <a href="#post-datasets-credentials" id="post-datasets-credentials"></a>

Returns temporary, read-only S3 credentials for the datasets. No body.

```json
{
  "access_key_id": "…",
  "secret_access_key": "…",
  "session_token": "…",
  "expiration": "2026-10-27T13:00:00Z",
  "s3_uri": "s3://data.termina.technology/datasets/",
  "user_s3_uri": "s3://data.termina.technology/userdata/<your_key_name>/datasets/",
  "region": "<aws_region>"
}
```

* `s3_uri` holds the prebuilt datasets; `user_s3_uri` your own jobs' output and reroute results.
* `user_s3_uri` is absent when your key's name can't be used as a storage path. Then you can read only the prebuilt datasets: contact us.
* `region` and `expiration` can be `null`.
* The credentials last an hour. Request new ones after `expiration`.

## Usage

### GET /usage <a href="#get-usage" id="get-usage"></a>

Your usage over a window, per UTC day.

* `since`, `until` (optional): RFC 3339 times. They default to the server's window. The report starts at the UTC midnight at or before `since`.

The response has `api_key_name`, `since`, `until`, `sessions` (`started`, `completed`, `failed`), `compute` (`executed_slot_count`, `session_duration_ms`), `daily` (the same counts per day), and, when they apply, `bundle_builds` and `pricing`. The full shape is [`UsageReport`](https://docs.rs/simulator-api/latest/simulator_api/usage/struct.UsageReport.html).

### GET /usage/sessions <a href="#get-usage-sessions" id="get-usage-sessions"></a>

Every session that ended in a window, one page at a time.

* `since`, `until` (optional): as for `/usage`.
* `cursor` (optional): the previous page's `next_cursor`.

```json
{
  "since": "2026-10-01T00:00:00Z",
  "until": "2026-10-27T12:00:00Z",
  "sessions": [
    { "session_id": "backtest_…", "session_name": "sol-usdc-baseline", "started_at": "…", "ended_at": "…", "executed_slot_count": 100, "session_duration_ms": 61234 }
  ],
  "next_cursor": "…"
}
```

When `next_cursor` is present, request the next page with it, and with the `since` and `until` this page returned. Pages aren't in time order: sort by `ended_at`.
