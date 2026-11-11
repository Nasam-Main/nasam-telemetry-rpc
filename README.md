# nasam-telemetry-rpc

The Protocol Buffers contract of the NASAM telemetry service: the read-only
gRPC API through which the Nasam API reads stored telemetry from the tracking
stack.

```
Nasam API (client) ──gRPC──► telemetry-api in nasam-tracking (server) ──► TimescaleDB
```

This repository holds the contract and nothing else. There is no generated
code, no client and no package to install: each service generates its own code
from the `.proto` at a pinned tag.

## Contract

`proto/nasam/telemetry/v1/telemetry.proto`, package `nasam.telemetry.v1`,
service `TelemetryService`:

| RPC | Kind | Answers |
|---|---|---|
| `StreamPlaybackPoints` | server stream | the valid positions devices recorded while fitted to an asset, with when the server received each, oldest first, at most 1,000 per message |
| `GetSummaryTotals` | unary | distance, engine and moving time per asset and day, from the half-hourly summaries |
| `GetRawTotals` | unary | the same totals worked out from raw records, for pieces of a half hour |
| `GetFixes` | unary | each asset's first or last valid position fix, optionally only those at a minimum speed |
| `GetSummaryOdometer` | unary | the first and last odometer reading per range, with reset and jump flags, from the half-hourly summaries |
| `GetRawOdometer` | unary | the same spans worked out from raw records, for pieces of a half hour |
| `GetLatestOdometer` | unary | each probe's latest odometer reading, from raw records near its end, else the summaries |
| `GetIntervals` | unary | moving, idle, parked, ignition-on, stop, door-open, speeding and inside-an-area intervals per range, from raw records |

Conventions:

- **Auth:** every call carries `authorization: Bearer <token>` metadata, a
  secret the two services share.
- **Scope:** the caller decides which devices its user may see. Requests name a
  tenant and the devices and time ranges to read; the service reads only those.
- **Times** are UTC `google.protobuf.Timestamp`s, and every range is half-open
  (`from <= t < to`).
- **Unset means not reported.** Nullable values are `optional` fields or unset
  messages, never zero.
- **Limits:** at most 2,000 ranges, probes or fittings per request, and 100 areas.
- **Errors** are gRPC statuses: `INVALID_ARGUMENT`, `UNAUTHENTICATED`,
  `UNAVAILABLE`, `DEADLINE_EXCEEDED`.
- **Deadlines:** the client sets one per call; the server stops the query when
  it passes.

## Using it

Pin a release tag and generate with [buf](https://buf.build) straight from this
repository, then commit the output in your own repository:

```bash
buf generate "https://github.com/Nasam-Main/nasam-telemetry-rpc.git#tag=v0.4.0"
```

The `buf.gen.yaml` with the plugins and options belongs to the consumer. The
proto sets no `go_package`; a Go consumer gives one in its `buf.gen.yaml`
through managed mode.

| Consumer | Role | Generates |
|---|---|---|
| `nasam-tracking` | server | Go (`protoc-gen-go`, `protoc-gen-go-grpc`) |
| `nasam-v2-api` | client | TypeScript (`ts-proto`, grpc-js services) |

## Changing it

Within `v1`, only add: new RPCs, new messages, new fields with new numbers.
Never renumber, retype, rename or remove a field. CI runs `buf breaking`
against the previous release tag and fails on anything else.

Before pushing:

```bash
buf lint
```

```bash
buf format -w
```

Roll a change out in this order: tag here, then the server (`nasam-tracking`),
then the client (`nasam-v2-api`), so the server always understands what the
client sends.

## Release

Push a `v<version>` tag. After CI passes on the tag, the workflow sends a
`telemetry-rpc-released` repository dispatch, with the tag, to each consumer,
which opens a pull request that moves its pin and regenerates.
`CONSUMER_REPOS_TOKEN` is optional: a fine-grained GitHub token limited to the
consumer repositories, with Contents: Read and write (what triggering a
repository dispatch needs). Without it the dispatch is skipped and each
consumer finds the new tag in its daily scheduled check of this repository, so
a release reaches them within a day instead of at once.

This repository is public so consumers can read it without credentials. It
must never contain credentials, infrastructure details, customer data, SQL or
server implementation.
