# Extensions

Extensions attach implementation-specific behavior to a standard policy. Core
matching and keep/transform semantics are unchanged — the engine matches
telemetry with the policy's `log`/`metric`/`trace` target, then hands matched
records to the handler registered for the extension `type`.

Extensions are generic: `type`, `version`, and an opaque `config`. The engine
does not interpret `config` — it routes matched records to the handler for
`type`, which parses `config` itself.

Destinations are **not** defined inside the policy. An extension target (an S3
bucket, a downstream OTLP endpoint, etc.) is pre-configured — either locally on
the implementation or broadcast from a provider via
`SyncResponse.extension_configs`. A routing extension's `config` references the
target by `(kind, name)`; the reference lives inside `config`, not as a
first-class field.

See the [Extensions section of the spec](../../spec.md#extensions) for the full
rules.

## This Example

- `dump-waste-to-s3.yaml` — sample checkout errors to `0.01%` and, via
  `mode: dropped`, route the sampled-out records (the "waste") to the
  pre-configured `s3` target named `eu-bucket`.

The matching `eu-bucket` target (with its region/bucket/prefix config) is
configured out of band; the policy only names it. A client advertises which
targets it can reach via `ClientMetadata.supported_extensions`, so a provider
only sends policies pointing at reachable targets.
