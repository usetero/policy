# Policy Specification

Status: Draft

## Abstract

This document specifies the structure and semantics of telemetry policies. A
policy is an atomic, portable rule for processing telemetry data. Policies are
designed for implementation across the telemetry ecosystem using OpenTelemetry's
data model.

## Overview

A policy declares a single intent: match specific telemetry and apply an action.
Policies are independent units that do not reference each other and do not
depend on execution order. This independence enables parallel evaluation and
scaling to large policy sets without performance degradation.

Policies use the
[OpenTelemetry data model](https://opentelemetry.io/docs/specs/otel/overview/)
for field references, ensuring portability across any runtime that implements
this specification.

## Design Principles

This specification follows the
[OpenTelemetry Specification Principles](https://opentelemetry.io/docs/specs/otel/specification-principles/):

- **User Driven**: Policies address real-world telemetry processing needs.
- **General**: The specification defines behavior, not implementation details.
- **Stable**: Changes preserve backward compatibility.
- **Consistent**: Concepts apply uniformly across telemetry signals.
- **Simple**: The policy model is minimal and unambiguous.

Additionally, policies adhere to these constraints:

- **Atomic**: A policy serves one intent with one matcher and one action set.
- **Self-contained**: A policy MUST NOT reference or depend on other policies.
- **Fail-open**: Policy evaluation failures MUST NOT cause telemetry loss.
- **Idempotent**: Applying the same policy multiple times produces the same
  result.

## Policy Structure

### Required Fields

A policy MUST contain the following fields:

| Field  | Type   | Description                       |
| ------ | ------ | --------------------------------- |
| `id`   | string | Unique identifier for the policy. |
| `name` | string | Human-readable name.              |

### Optional Fields

A policy MAY contain the following fields:

| Field                   | Type       | Default | Description                                            |
| ----------------------- | ---------- | ------- | ------------------------------------------------------ |
| `description`           | string     | empty   | Explanation of the policy's purpose.                   |
| `enabled`               | boolean    | `true`  | Whether the policy is active.                          |
| `created_at_unix_nano`  | fixed64    | -       | Creation timestamp in Unix epoch nanoseconds.          |
| `modified_at_unix_nano` | fixed64    | -       | Last modification timestamp in Unix epoch nanoseconds. |
| `labels`                | KeyValue[] | empty   | Metadata labels for routing and organization.          |

### Target

A policy MUST specify exactly one target. The target defines which telemetry
signal the policy applies to and contains the matching and action configuration.

Currently defined targets:

| Target   | Description                                               |
| -------- | --------------------------------------------------------- |
| `log`    | Log record processing. See [Log Target](#log-target).     |
| `metric` | Metric processing. See [Metric Target](#metric-target).   |
| `trace`  | Trace/span processing. See [Trace Target](#trace-target). |

## Log Target

The `log` target defines matching criteria and actions for log records.

### Structure

```
LogTarget {
  match:     LogMatcher[]   // REQUIRED, at least one matcher
  keep:      string         // OPTIONAL, defaults to "all"
  transform: LogTransform   // OPTIONAL
}
```

A log target MUST contain at least one matcher. A log target MUST specify either
a `keep` value other than `"all"` or a `transform`, or both.

### Log Matching

Matchers identify which log records a policy applies to. Multiple matchers are
combined with AND logic: all matchers MUST match for the policy to apply.

#### LogMatcher Structure

```
LogMatcher {
  field:  <field selector>  // REQUIRED, exactly one
  match:  <match type>      // REQUIRED, exactly one
  negate: boolean           // OPTIONAL, defaults to false
}
```

#### Field Selection

A matcher MUST specify exactly one field selector:

| Selector             | Type          | Description                              |
| -------------------- | ------------- | ---------------------------------------- |
| `log_field`          | LogField enum | Well-known log record field.             |
| `log_attribute`      | AttributePath | Log record attribute by path.            |
| `resource_attribute` | AttributePath | Resource attribute by path.              |
| `scope_attribute`    | AttributePath | Instrumentation scope attribute by path. |

##### AttributePath

Attribute selectors use an `AttributePath` to specify which attribute to access.
The path is an array of string segments, where each segment represents a key to
traverse into nested maps.

```yaml
# Flat attribute access (single segment)
log_attribute: ["user_id"]

# Nested attribute access (multiple segments)
log_attribute: ["http", "request", "method"]
```

For example, given an attribute structure:

```
Attributes: {
    "http": {
        "request": {
            "method": "POST",
            "url": "/api/users"
        }
    },
    "user_id": "u123"
}
```

- `["user_id"]` accesses the flat `user_id` attribute
- `["http", "request", "method"]` traverses to access `"POST"`

###### Unmarshaling

The proto definition wraps the path in an `AttributePath` message with a `path`
field. This wrapper is required because proto3 does not allow `repeated` fields
directly inside a `oneof`. However, for ergonomic policy authoring,
implementations MUST accept both the canonical proto form and shorthand forms
when unmarshaling from YAML/JSON:

```yaml
# Canonical (proto-native) - MUST be supported
log_attribute:
  path: ["http", "method"]

# Shorthand array - MUST be supported
log_attribute: ["http", "method"]

# Shorthand string (single-segment only) - MUST be supported
log_attribute: "user_id"
```

When marshaling, implementations SHOULD use the shorthand array form for cleaner
output.

##### LogField Enum Values

| Value                           | Description                                          |
| ------------------------------- | ---------------------------------------------------- |
| `LOG_FIELD_BODY`                | The log message body.                                |
| `LOG_FIELD_SEVERITY_TEXT`       | Severity as string (e.g., "DEBUG", "INFO", "ERROR"). |
| `LOG_FIELD_TRACE_ID`            | Associated trace identifier.                         |
| `LOG_FIELD_SPAN_ID`             | Associated span identifier.                          |
| `LOG_FIELD_EVENT_NAME`          | Event name for event logs.                           |
| `LOG_FIELD_RESOURCE_SCHEMA_URL` | Resource schema URL.                                 |
| `LOG_FIELD_SCOPE_SCHEMA_URL`    | Scope schema URL.                                    |

#### Match Types

A matcher MUST specify exactly one match type:

| Type     | Value Type | Description                                                    |
| -------- | ---------- | -------------------------------------------------------------- |
| `exact`  | string     | Field value MUST equal the specified string exactly.           |
| `regex`  | string     | Field value MUST match the regular expression.                 |
| `exists` | boolean    | If `true`, field MUST exist. If `false`, field MUST NOT exist. |

Regular expressions MUST use
[RE2 syntax](https://github.com/google/re2/wiki/Syntax) for cross-implementation
consistency.

#### Negation

If `negate` is `true`, the match result is inverted. A matcher that would match
does not match, and vice versa.

### Keep

The `keep` field controls whether matching telemetry survives processing. It
unifies dropping, sampling, and rate limiting into a single concept.

| Value    | Description                                       |
| -------- | ------------------------------------------------- |
| `"all"`  | Keep all matching telemetry. This is the default. |
| `"none"` | Drop all matching telemetry.                      |
| `"N%"`   | Keep N percent of matching telemetry (0-100).     |
| `"N/s"`  | Keep at most N records per second.                |
| `"N/m"`  | Keep at most N records per minute.                |

Implementations MUST support `"all"` and `"none"`. Implementations SHOULD
support percentage-based sampling. Implementations MAY support rate limiting.

When multiple policies match the same telemetry with different `keep` values,
the most restrictive value MUST be applied:

1. `"none"` takes precedence over any other value.
2. Lower percentages take precedence over higher percentages.
3. Rate limits are evaluated independently per policy.

### Log Transform

The `transform` field specifies modifications to apply to log records that
survive the keep stage.

#### Structure

```
LogTransform {
  remove: LogRemove[]  // OPTIONAL
  redact: LogRedact[]  // OPTIONAL
  rename: LogRename[]  // OPTIONAL
  add:    LogAdd[]     // OPTIONAL
}
```

#### Execution Order

Transform operations MUST execute in the following order:

1. **Remove**: Delete specified fields.
2. **Redact**: Mask specified field values.
3. **Rename**: Change field names.
4. **Add**: Insert new fields.

This ordering is normative. Implementations MUST NOT reorder these stages.

#### LogRemove

Removes a field from the log record.

```
LogRemove {
  field: <field selector>  // REQUIRED, exactly one
}
```

The field selector uses the same options as LogMatcher: `log_field`,
`log_attribute`, `resource_attribute`, or `scope_attribute`.

If the specified field does not exist, the operation MUST be a no-op.

#### LogRedact

Masks a field value with a replacement string.

```
LogRedact {
  field:       <field selector>  // REQUIRED, exactly one
  replacement: string            // OPTIONAL, defaults to "[REDACTED]"
}
```

If the specified field does not exist, the operation MUST be a no-op.

#### LogRename

Changes a field's key to a new name.

```
LogRename {
  from:   <field selector>  // REQUIRED, exactly one
  to:     string            // REQUIRED
  upsert: boolean           // OPTIONAL, defaults to false
}
```

If the source field does not exist, the operation MUST be a no-op.

If `upsert` is `false` and the target field already exists, the operation MUST
be a no-op. If `upsert` is `true`, the target field MUST be overwritten.

#### LogAdd

Inserts a new field with a specified value.

```
LogAdd {
  field:  <field selector>  // REQUIRED, exactly one
  value:  string            // REQUIRED
  upsert: boolean           // OPTIONAL, defaults to false
}
```

If `upsert` is `false` and the field already exists, the operation MUST be a
no-op. If `upsert` is `true`, the field MUST be overwritten.

## Metric Target

The `metric` target defines matching criteria and actions for metrics.

### Structure

```
MetricTarget {
  match: MetricMatcher[]  // REQUIRED, at least one matcher
  keep:  boolean          // REQUIRED
}
```

A metric target MUST contain at least one matcher and a `keep` value.

### Metric Matching

Matchers identify which metrics a policy applies to. Multiple matchers are
combined with AND logic: all matchers MUST match for the policy to apply.

#### MetricMatcher Structure

```
MetricMatcher {
  field:  <field selector>  // REQUIRED, exactly one
  match:  <match type>      // REQUIRED, exactly one (except for metric_type)
  negate: boolean           // OPTIONAL, defaults to false
}
```

#### Field Selection

A matcher MUST specify exactly one field selector:

| Selector              | Type             | Description                              |
| --------------------- | ---------------- | ---------------------------------------- |
| `metric_field`        | MetricField enum | Well-known metric field.                 |
| `datapoint_attribute` | AttributePath    | Data point attribute by path.            |
| `resource_attribute`  | AttributePath    | Resource attribute by path.              |
| `scope_attribute`     | AttributePath    | Instrumentation scope attribute by path. |
| `metric_type`         | MetricType enum  | Metric type (implicit equality match).   |

See [AttributePath](#attributepath) for path syntax.

##### MetricField Enum Values

| Value                              | Description             |
| ---------------------------------- | ----------------------- |
| `METRIC_FIELD_NAME`                | The metric name.        |
| `METRIC_FIELD_DESCRIPTION`         | The metric description. |
| `METRIC_FIELD_UNIT`                | The metric unit.        |
| `METRIC_FIELD_RESOURCE_SCHEMA_URL` | Resource schema URL.    |
| `METRIC_FIELD_SCOPE_SCHEMA_URL`    | Scope schema URL.       |

##### MetricType Enum Values

| Value                               | Description                   |
| ----------------------------------- | ----------------------------- |
| `METRIC_TYPE_GAUGE`                 | Gauge metric.                 |
| `METRIC_TYPE_SUM`                   | Sum metric.                   |
| `METRIC_TYPE_HISTOGRAM`             | Histogram metric.             |
| `METRIC_TYPE_EXPONENTIAL_HISTOGRAM` | Exponential histogram metric. |
| `METRIC_TYPE_SUMMARY`               | Summary metric.               |

#### Match Types

A matcher MUST specify exactly one match type (except when using `metric_type`,
which performs implicit equality):

| Type     | Value Type | Description                                                    |
| -------- | ---------- | -------------------------------------------------------------- |
| `exact`  | string     | Field value MUST equal the specified string exactly.           |
| `regex`  | string     | Field value MUST match the regular expression.                 |
| `exists` | boolean    | If `true`, field MUST exist. If `false`, field MUST NOT exist. |

Regular expressions MUST use
[RE2 syntax](https://github.com/google/re2/wiki/Syntax) for cross-implementation
consistency.

#### Negation

If `negate` is `true`, the match result is inverted. A matcher that would match
does not match, and vice versa.

### Keep

The `keep` field is a boolean that controls whether matching metrics survive
processing:

| Value   | Description                |
| ------- | -------------------------- |
| `true`  | Keep all matching metrics. |
| `false` | Drop all matching metrics. |

When multiple policies match the same metric, if any policy specifies
`keep: false`, the metric MUST be dropped.

## Trace Target

The `trace` target defines matching criteria and probabilistic sampling actions
for traces/spans.

### Structure

```
TraceTarget {
  match: TraceMatcher[]       // REQUIRED, at least one matcher
  keep:  TraceSamplingConfig  // REQUIRED
}
```

A trace target MUST contain at least one matcher and a `keep` configuration.

### Trace Matching

Matchers identify which spans a policy applies to. Multiple matchers are
combined with AND logic: all matchers MUST match for the policy to apply.

#### TraceMatcher Structure

```
TraceMatcher {
  field:  <field selector>  // REQUIRED, exactly one
  match:  <match type>      // REQUIRED, exactly one (except for span_kind/span_status)
  negate: boolean           // OPTIONAL, defaults to false
}
```

#### Field Selection

A matcher MUST specify exactly one field selector:

| Selector             | Type                | Description                                      |
| -------------------- | ------------------- | ------------------------------------------------ |
| `trace_field`        | TraceField enum     | Well-known span field.                           |
| `span_attribute`     | AttributePath       | Span attribute by path.                          |
| `resource_attribute` | AttributePath       | Resource attribute by path.                      |
| `scope_attribute`    | AttributePath       | Instrumentation scope attribute by path.         |
| `span_kind`          | SpanKind enum       | Span kind (implicit equality match).             |
| `span_status`        | SpanStatusCode enum | Span status code (implicit equality match).      |
| `event_name`         | string              | Event name (matches if span contains the event). |
| `event_attribute`    | AttributePath       | Event attribute path (matches if present).       |
| `link_trace_id`      | string              | Link trace ID (matches if span has link).        |

See [AttributePath](#attributepath) for path syntax.

##### TraceField Enum Values

| Value                             | Description                    |
| --------------------------------- | ------------------------------ |
| `TRACE_FIELD_NAME`                | The span name.                 |
| `TRACE_FIELD_TRACE_ID`            | The trace identifier.          |
| `TRACE_FIELD_SPAN_ID`             | The span identifier.           |
| `TRACE_FIELD_PARENT_SPAN_ID`      | The parent span identifier.    |
| `TRACE_FIELD_TRACE_STATE`         | The W3C tracestate.            |
| `TRACE_FIELD_RESOURCE_SCHEMA_URL` | Resource schema URL.           |
| `TRACE_FIELD_SCOPE_SCHEMA_URL`    | Scope schema URL.              |
| `TRACE_FIELD_SCOPE_NAME`          | Instrumentation scope name.    |
| `TRACE_FIELD_SCOPE_VERSION`       | Instrumentation scope version. |

##### SpanKind Enum Values

| Value                | Description                              |
| -------------------- | ---------------------------------------- |
| `SPAN_KIND_INTERNAL` | Internal operation (default).            |
| `SPAN_KIND_SERVER`   | Server-side handling of a request.       |
| `SPAN_KIND_CLIENT`   | Client-side request to a remote service. |
| `SPAN_KIND_PRODUCER` | Producer of an asynchronous message.     |
| `SPAN_KIND_CONSUMER` | Consumer of an asynchronous message.     |

##### SpanStatusCode Enum Values

| Value                          | Description                       |
| ------------------------------ | --------------------------------- |
| `SPAN_STATUS_CODE_UNSPECIFIED` | Status not set.                   |
| `SPAN_STATUS_CODE_OK`          | Operation completed successfully. |
| `SPAN_STATUS_CODE_ERROR`       | Operation contained an error.     |

#### Match Types

A matcher MUST specify exactly one match type (except when using `span_kind` or
`span_status`, which perform implicit equality):

| Type     | Value Type | Description                                                    |
| -------- | ---------- | -------------------------------------------------------------- |
| `exact`  | string     | Field value MUST equal the specified string exactly.           |
| `regex`  | string     | Field value MUST match the regular expression.                 |
| `exists` | boolean    | If `true`, field MUST exist. If `false`, field MUST NOT exist. |

Regular expressions MUST use
[RE2 syntax](https://github.com/google/re2/wiki/Syntax) for cross-implementation
consistency.

#### Negation

If `negate` is `true`, the match result is inverted. A matcher that would match
does not match, and vice versa.

### Keep (Probabilistic Sampling)

The `keep` field configures probabilistic sampling for traces. This follows the
[OpenTelemetry Probability Sampling specification](https://opentelemetry.io/docs/specs/otel/trace/tracestate-probability-sampling/).

#### TraceSamplingConfig Structure

```
TraceSamplingConfig {
  percentage:         float   // REQUIRED, 0-100
  mode:               string  // OPTIONAL, defaults to "hash_seed"
  sampling_precision: integer // OPTIONAL, defaults to 4
  hash_seed:          integer // OPTIONAL, defaults to 0
  fail_closed:        boolean // OPTIONAL, defaults to true
}
```

#### Configuration Fields

| Field                | Type    | Default     | Description                                                         |
| -------------------- | ------- | ----------- | ------------------------------------------------------------------- |
| `percentage`         | float   | -           | Sampling percentage (0-100). >=100 keeps all, 0 drops all.          |
| `mode`               | string  | `hash_seed` | Sampling mode. One of `hash_seed`, `proportional`, or `equalizing`. |
| `sampling_precision` | integer | 4           | Hex digits for threshold encoding (1-14).                           |
| `hash_seed`          | integer | 0           | Hash seed for deterministic sampling.                               |
| `fail_closed`        | boolean | true        | If true, reject items on sampling errors.                           |

#### Sampling Modes

| Mode           | Description                                                                                   |
| -------------- | --------------------------------------------------------------------------------------------- |
| `hash_seed`    | Uses hash of trace ID with seed for deterministic sampling. Default mode.                     |
| `proportional` | Respects existing sampling probability in tracestate. Adjusts to achieve target overall rate. |
| `equalizing`   | Balances sampling across sources by preferentially sampling higher-rate spans.                |

##### hash_seed Use Cases

The `hash_seed` mode uses an FNV hash of the trace ID combined with the
configured seed to make deterministic sampling decisions. This mode is useful
when you need consistent sampling across multiple collectors behind a load
balancer—all collectors with the same `hash_seed` will make identical decisions
for the same trace ID. Different tiers can use different seeds to apply
additional sampling. See the
[OTel Collector hash_seed documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/probabilisticsamplerprocessor#hash-seed-use-cases)
for details.

##### proportional Use Cases

The `proportional` mode provides predictable output ratios regardless of
upstream sampling configuration. If a collector is configured for 25%
proportional sampling, it will output approximately one span for every four
received, regardless of how they were sampled upstream. This mode follows
OpenTelemetry and W3C Trace Context Level 2 specifications. Use this when you
need reliable throughput reduction or uniform downstream enforcement across
mixed client configurations. See the
[OTel Collector proportional documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/probabilisticsamplerprocessor#proportional-use-cases)
for details.

##### equalizing Use Cases

The `equalizing` mode is designed for deployments with mixed sampling
configurations across components. For example, if in-house services sample at
10% but third-party services are unsampled, an equalizing sampler at the
collector can apply uniform 10% sampling: already-sampled telemetry passes
through unchanged while unsampled telemetry gets sampled at the configured rate.
Unlike proportional mode, equalizing considers prior sampling decisions and only
updates the threshold when beneficial. See the
[OTel Collector equalizing documentation](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/probabilisticsamplerprocessor#equalizing-use-cases)
for details.

#### Tracestate Handling

Implementations MUST follow the
[OpenTelemetry tracestate handling specification](https://opentelemetry.io/docs/specs/otel/trace/tracestate-handling/#sampling-threshold-value-th)
to enable multi-stage sampling:

- The sampling threshold MUST be encoded in the `th` sub-key of the `ot`
  tracestate entry.
- The threshold is a 56-bit value encoded as 1-14 lowercase hexadecimal digits
  with trailing zeros removed.
- Sampling decisions are made by comparing a randomness value (R) against the
  rejection threshold (T): if R >= T, keep the span.
- Downstream samplers can only increase thresholds (decrease probability), never
  decrease them.

## Policy Stages

Policies execute in two fixed stages:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   Telemetry ──▶ Match ──┬──▶ Keep ──▶ Transform ──▶ Out         │
│                         │                                       │
│                         │            ┌─────────────────┐        │
│                         │            │ 1. Remove       │        │
│                         │            │ 2. Redact       │        │
│                         │            │ 3. Rename       │        │
│                         │            │ 4. Add          │        │
│                         │            └─────────────────┘        │
│                         │                                       │
│                         └── (policies matched in parallel)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Stage 1: Keep

All matching policies contribute their `keep` values. The runtime evaluates
these values and applies the most restrictive result. If telemetry is dropped or
sampled out, processing stops.

### Stage 2: Transform

All matching policies contribute their transform operations. Operations execute
in the defined order: remove → redact → rename → add. Within each operation
type, if multiple policies target the same field, the result is
implementation-defined but MUST be deterministic.

Implementations MUST process policies in a consistent order (e.g.,
alphabetically by ID) to ensure reproducible results.

## Runtime Requirements

### Evaluation

Implementations MAY evaluate policies concurrently. The independence of policies
enables parallel matching without coordination.

### Error Handling

Implementations MUST be fail-open:

- If a policy fails to parse, it MUST be skipped. Other policies MUST continue
  to execute.
- If a policy fails to evaluate (e.g., invalid regex at runtime), the telemetry
  MUST pass through unmodified by that policy.
- Policy failures MUST NOT cause telemetry loss.

Implementations SHOULD log policy evaluation errors for debugging.

### Disabled Policies

Policies with `enabled: false` MUST NOT be evaluated. Implementations MUST treat
disabled policies as if they do not exist.

## YAML Representation

While the canonical format is Protocol Buffers, policies MAY be represented in
YAML for human authoring. The YAML structure MUST map directly to the protobuf
schema.

Example policy in YAML:

```yaml
id: drop-checkout-debug-logs
name: Drop checkout debug logs
description: Remove debug-level logs from checkout-api to reduce volume
enabled: true
log:
  match:
    - resource_attribute: ["service.name"]
      exact: checkout-api
    - log_field: LOG_FIELD_SEVERITY_TEXT
      exact: DEBUG
  keep: none
```

Example with transform:

```yaml
id: redact-payment-pii
name: Redact PII from payment service
log:
  match:
    - resource_attribute: ["service.name"]
      exact: payment-api
  transform:
    remove:
      - log_attribute: ["user.password"]
    redact:
      - log_attribute: ["user.email"]
      - log_attribute: ["user.phone"]
        replacement: "[PHONE]"
    add:
      - log_attribute: ["sanitized"]
        value: "true"
        upsert: true
```

Example with nested attribute access:

```yaml
id: redact-http-auth-header
name: Redact HTTP authorization headers
log:
  match:
    - log_attribute: ["http", "request", "headers", "authorization"]
      exists: true
  transform:
    redact:
      - log_attribute: ["http", "request", "headers", "authorization"]
```

Example metric policy:

```yaml
id: drop-debug-metrics
name: Drop debug metrics
metric:
  match:
    - metric_field: METRIC_FIELD_NAME
      regex: "^debug\\."
  keep: false
```

Example trace policy with probabilistic sampling:

```yaml
id: sample-database-spans
name: Sample database spans at 5%
trace:
  match:
    - span_attribute: ["db.system"]
      exists: true
  keep:
    percentage: 5.0
    mode: equalizing
```

Example trace policy keeping all error spans:

```yaml
id: keep-error-spans
name: Keep all error spans
trace:
  match:
    - span_status: ERROR
  keep:
    percentage: 100.0
```

## Conformance

An implementation conforms to this specification if it:

1. Correctly parses valid policies as defined in this document.
2. Evaluates matchers according to the specified semantics.
3. Applies keep logic with proper precedence rules.
4. Executes transform operations in the specified order.
5. Maintains fail-open behavior for all error conditions.
6. Respects the `enabled` field.

Implementations MAY support a subset of features (e.g., omit rate limiting) but
MUST clearly document unsupported features.

## References

- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)
- [OpenTelemetry Log Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [OpenTelemetry Trace API](https://opentelemetry.io/docs/specs/otel/trace/api/)
- [OpenTelemetry Probability Sampling](https://opentelemetry.io/docs/specs/otel/trace/tracestate-probability-sampling/)
- [OpenTelemetry Tracestate Handling](https://opentelemetry.io/docs/specs/otel/trace/tracestate-handling/)
- [RE2 Regular Expression Syntax](https://github.com/google/re2/wiki/Syntax)
- [RFC 2119 - Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)
- [Policy OTEP](https://github.com/open-telemetry/opentelemetry-specification/pull/4738)
