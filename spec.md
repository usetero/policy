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

| Value                           | Type   | Description                                          |
| ------------------------------- | ------ | ---------------------------------------------------- |
| `LOG_FIELD_BODY`                | string | The log message body.                                |
| `LOG_FIELD_SEVERITY_TEXT`       | string | Severity as string (e.g., "DEBUG", "INFO", "ERROR"). |
| `LOG_FIELD_TRACE_ID`            | bytes  | Associated trace identifier. See [Bytes and Identifier Fields](#bytes-and-identifier-fields). |
| `LOG_FIELD_SPAN_ID`             | bytes  | Associated span identifier. See [Bytes and Identifier Fields](#bytes-and-identifier-fields). |
| `LOG_FIELD_EVENT_NAME`          | string | Event name for event logs.                           |
| `LOG_FIELD_RESOURCE_SCHEMA_URL` | string | Resource schema URL.                                 |
| `LOG_FIELD_SCOPE_SCHEMA_URL`    | string | Scope schema URL.                                    |

#### Match Types

A matcher MUST specify exactly one match type:

| Type          | Value Type | Description                                                    |
| ------------- | ---------- | -------------------------------------------------------------- |
| `exact`       | string     | String field value MUST equal the specified string exactly.    |
| `regex`       | string     | String field value MUST match the regular expression.          |
| `exists`      | boolean    | If `true`, field MUST exist. If `false`, field MUST NOT exist. |
| `starts_with` | string     | String field value MUST begin with the specified literal string. |
| `ends_with`   | string     | String field value MUST end with the specified literal string. |
| `contains`    | string     | String field value MUST contain the specified literal substring. |
| `equals`      | Value      | Non-string field value MUST equal the typed value. See [Typed and Comparison Matching](#typed-and-comparison-matching). |
| `gt`          | NumericValue | Numeric field value MUST be greater than the value.          |
| `gte`         | NumericValue | Numeric field value MUST be greater than or equal to the value. |
| `lt`          | NumericValue | Numeric field value MUST be less than the value.             |
| `lte`         | NumericValue | Numeric field value MUST be less than or equal to the value. |

Regular expressions MUST use
[RE2 syntax](https://github.com/google/re2/wiki/Syntax) for cross-implementation
consistency.

#### Typed and Comparison Matching

The `exact`, `regex`, `starts_with`, `ends_with`, and `contains` match types
operate on the field's **string** value. For most non-string values (bool, int,
double) they never match; use the typed `equals` matcher or the numeric
comparison matchers `gt`, `gte`, `lt`, and `lte` instead. Bytes-typed fields are
the exception and have dedicated handling — see
[Bytes and Identifier Fields](#bytes-and-identifier-fields).

These matchers carry a typed value rather than a string. `equals` uses `Value`,
which holds any non-string scalar; the comparison matchers use `NumericValue`,
which holds only a number:

```
Value {
  // exactly one of:
  bool_value:   bool
  int_value:    int64
  double_value: double
  bytes_value:  bytes    // raw bytes (base64 in JSON)
  hex_value:    string   // bytes as a hex string (readable; see below)
}

NumericValue {
  // exactly one of:
  int_value:    int64
  double_value: double
}
```

Two distinct types are used deliberately: because the comparison matchers take a
`NumericValue`, comparing against a non-numeric value (a bool or bytes) is
unrepresentable in the schema rather than something that must be rejected during
compilation. `Value` has no string variant for the same reason — string
equality is expressed with `exact`. `int_value` preserves full 64-bit precision
for large integer fields (for example, nanosecond timestamps) that a `double`
cannot represent exactly.

> **Future direction:** `exact` is expected to be deprecated. Once `Value` gains
> a string variant, all equality — string and non-string — will be expressed
> through a single `equals` matcher, and `exact` will be retained only for
> backward compatibility. New policies SHOULD treat `exact` and `equals` as the
> same concept differing only by the value's type.

**Semantics:**

- `equals` matches if and only if the field value has the same type and value as
  the supplied `Value`. Integer and floating-point values are compared in a
  single numeric domain, so an `int_value` matcher MAY match a `double` field
  with an equal numeric value and vice versa. All other type pairings (for
  example, an `int_value` matcher against a string field) MUST NOT match.
- `gt`, `gte`, `lt`, and `lte` perform numeric comparison against `int` and
  `double` field values. A non-numeric field value MUST NOT match.
- A type mismatch is a non-match, never a runtime error (fail-open).
- `case_insensitive` has no effect on `equals` or the comparison matchers; it
  applies only to the string match types.
- `negate` inverts the result, as with any matcher.

**Authoring:** As with [AttributePath](#attributepath), implementations MUST
accept both the canonical proto form and scalar shorthand when unmarshaling from
YAML/JSON. For shorthand, the literal's type determines the `Value` variant:

```yaml
# Shorthand — type inferred from the literal
- log_attribute: ["http.response.status_code"]
  gte: 500                 # int
- log_attribute: ["sampling.ratio"]
  lt: 0.5                  # double
- log_attribute: ["deprecated"]
  equals: true             # bool

# Canonical proto form
- log_attribute: ["http.response.status_code"]
  equals:
    int_value: 200
```

Bytes are supplied either as the proto-native base64 `bytes_value` or, more
readably, as a hex string in `hex_value`; see
[Bytes and Identifier Fields](#bytes-and-identifier-fields).

**Validation:** A string value supplied to `equals` (use `exact` instead), and a
bool or bytes value supplied to a comparison matcher, are not representable in
`Value`/`NumericValue` and MUST be rejected when unmarshaling. Per
[Compilation Errors](#compilation-errors), a policy is also invalid if a
`hex_value` is not valid hexadecimal (non-hex characters or an odd number of
digits), or if the typed value is left unset (an empty `Value`/`NumericValue`).

#### Bytes and Identifier Fields

Some fields are byte sequences rather than strings. The well-known identifier
fields — `*_TRACE_ID`, `*_SPAN_ID`, and `*_PARENT_SPAN_ID` — are `bytes`, as is
any telemetry attribute whose value is a byte sequence. Each well-known field's
type is given in its enum table. The `link_trace_id` selector also carries a
trace id and follows the same rules.

**Canonical encoding.** Bytes are authored and rendered as **lowercase
hexadecimal** with an even number of digits. Hex is the canonical human encoding
because it is how identifiers appear throughout OpenTelemetry (for example, the
W3C `traceparent` header). Implementations MUST decode the hex literal to raw
bytes **once, at policy-compile time**, and store the decoded bytes; the bytes —
never the hex text — are what appear in the canonical protobuf form, so the
encoding cost is paid once per policy rather than once per telemetry record.

**Authoring.** A bytes-typed field is matched as follows:

```yaml
# Well-known identifier field: a plain string literal is decoded as hex,
# because the field's declared type is bytes (no decoration needed).
- trace_field: TRACE_FIELD_SPAN_ID
  exact: "8a3f0e1234567890"

# Arbitrary bytes attribute: the type is not known from the field, so the bytes
# value is supplied explicitly via equals.
- log_attribute: ["raw.token"]
  equals:
    hex_value: "deadbeef"      # hex string (readable)
- log_attribute: ["raw.token"]
  equals:
    bytes_value: "3q2+7w=="    # proto-native base64
```

`hex_value` and `bytes_value` are two encodings of the same bytes; an
implementation decodes whichever is set to raw bytes at compile time, so they
are interchangeable.

**Matching semantics on a bytes-typed field:**

- `exact` and `equals` decode the hex (or base64) literal once and compare
  `bytes == bytes`. Hex input is case-insensitive; the comparison is over raw
  bytes, so length and value are checked exactly.
- `exists` is unchanged.
- `regex`, `starts_with`, `ends_with`, and `contains` treat the field as a
  string, matching against its canonical lowercase-hex rendering. This keeps
  identifier prefix and substring matching deterministic.
- The numeric comparison matchers (`gt`, `gte`, `lt`, `lte`) do not apply to
  bytes.

**Type coercion and performance.** Implementations SHOULD always coerce between a
field's declared type and the matcher literal — for example, decoding a hex
string to bytes for a bytes-typed field — so a correctly authored policy matches
regardless of how the literal was written. For performance, policies SHOULD also
include a string match type where practical: implementations commonly optimize
string and regular-expression matching with a dedicated multi-pattern engine, and
a string matcher lets that fast path pre-filter records before a typed comparison
runs.

#### Case Insensitivity

If `case_insensitive` is `true`, the match is performed without regard to case.
This applies to all match types including `exact`, `regex`, `starts_with`,
`ends_with`, and `contains`.

#### Negation

If `negate` is `true`, the match result is inverted. A matcher that would match
does not match, and vice versa.

### Keep

The `keep` field controls whether matching telemetry survives processing. It
unifies dropping, sampling, and rate limiting into a single concept.

| Value    | Description                                                                     |
| -------- | ------------------------------------------------------------------------------- |
| `"all"`  | Keep all matching telemetry. This is the default.                               |
| `"none"` | Drop all matching telemetry.                                                    |
| `"N%"`   | Keep N percent of matching telemetry (0-100).                                   |
| `"N/s"`  | Keep at most N records per second (shorthand for `"N/1s"`).                     |
| `"N/m"`  | Keep at most N records per minute (shorthand for `"N/1m"`).                     |
| `"N/Ds"` | Keep at most N records per D-second window (`N` and `D` are positive integers). |
| `"N/Dm"` | Keep at most N records per D-minute window (`N` and `D` are positive integers). |

Implementations MUST support `"all"` and `"none"`. Implementations SHOULD
support percentage-based sampling. Implementations MAY support rate limiting.

For rate limiting, both `N` (limit) and `D` (window multiplier) MUST be positive
integers. Fractional values are invalid and MUST be rejected.

Examples: `"1/s"`, `"100/s"`, `"1/5s"`, `"1/300s"`, `"10/5m"`, `"1/m"`.

When multiple policies match the same telemetry with different `keep` values,
the most restrictive value MUST be applied:

1. `"none"` takes precedence over any other value.
2. Lower percentages take precedence over higher percentages.
3. Rate limits are evaluated independently per policy.

#### Consistent Sampling with sample_key

The `sample_key` field enables consistent sampling by specifying a field whose
value determines the sampling decision. When set, all logs with the same value
for the specified field receive the same keep/drop decision.

```
LogSampleKey {
  // Exactly one must be set:
  log_field:          LogField       // Simple fields (trace_id, span_id, etc.)
  log_attribute:      AttributePath  // Log record attribute
  resource_attribute: AttributePath  // Resource attribute
  scope_attribute:    AttributePath  // Scope attribute
}
```

This prevents "swiss cheese" sampling where related log lines (e.g., lifecycle
events for a single request) get sampled independently.

**Example:** A Rails request that produces 6 log lines:

```
Started GET "/products/123"
Processing by ProductsController#show
  Product Load (2.3ms) SELECT...
  Rendered products/show.html.erb
Completed 200 OK in 45ms
```

Without `sample_key`, at 10% sampling you get random combinations—some requests
with 2 logs, some with 4, some missing entirely. With
`sample_key: {log_attribute: "request_id"}` and `keep: "10%"`, you either get
all logs for a request or none.

**Implementation:** When `sample_key` is set and `keep` is a sampling value:

1. Extract the value of the specified field from the log
2. Hash the value deterministically (e.g., `hash(value) % 100 < percentage`)
3. Apply the same keep/drop decision to all logs with that key value

If `sample_key` is not set, sample each log independently (default behavior).

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

Masks a field value, or a targeted portion of a field value, with a replacement
string.

```
LogRedact {
  field:       <field selector>  // REQUIRED, exactly one
  replacement: string            // OPTIONAL, defaults to "[REDACTED]"
  regex:       string            // OPTIONAL
}
```

If the specified field does not exist, the operation MUST be a no-op.

If `regex` is not supplied, the operation MUST replace the field's entire value
with `replacement`.

If `regex` is supplied, implementations MUST use RE2 syntax and evaluate the
regular expression against the field's current string value:

- If the regular expression does not match, the operation MUST be a no-op.
- If the regular expression matches, the operation MUST replace all
  non-overlapping instances of the full regular expression match with
  `replacement`, not only the first match.
- Capture groups MUST NOT change the replacement range. Capture groups only
  provide values that MAY be referenced from the replacement string.

If `regex` is supplied and the field's current value is not a string, the
operation MUST be a no-op.

When `regex` is supplied, `replacement` is interpreted as a replacement
template. Implementations MUST support the following capture references:

| Syntax         | Description                        |
| -------------- | ---------------------------------- |
| `$0`           | The full regular expression match. |
| `$1`-`$99`     | Numbered capture groups.           |
| `${1}`-`${99}` | Numbered capture groups.           |
| `${name}`      | Named capture group.               |
| `$$`           | A literal dollar sign.             |

References to missing capture groups MUST expand to the empty string. To replace
an entire field value conditionally, use an anchored `regex` that matches the
full field value.

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

| Value                              | Type   | Description             |
| ---------------------------------- | ------ | ----------------------- |
| `METRIC_FIELD_NAME`                | string | The metric name.        |
| `METRIC_FIELD_DESCRIPTION`         | string | The metric description. |
| `METRIC_FIELD_UNIT`                | string | The metric unit.        |
| `METRIC_FIELD_RESOURCE_SCHEMA_URL` | string | Resource schema URL.    |
| `METRIC_FIELD_SCOPE_SCHEMA_URL`    | string | Scope schema URL.       |

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

| Type          | Value Type | Description                                                    |
| ------------- | ---------- | -------------------------------------------------------------- |
| `exact`       | string     | String field value MUST equal the specified string exactly.    |
| `regex`       | string     | String field value MUST match the regular expression.          |
| `exists`      | boolean    | If `true`, field MUST exist. If `false`, field MUST NOT exist. |
| `starts_with` | string     | String field value MUST begin with the specified literal string. |
| `ends_with`   | string     | String field value MUST end with the specified literal string. |
| `contains`    | string     | String field value MUST contain the specified literal substring. |
| `equals`      | Value      | Non-string field value MUST equal the typed value. See [Typed and Comparison Matching](#typed-and-comparison-matching). |
| `gt`          | NumericValue | Numeric field value MUST be greater than the value.          |
| `gte`         | NumericValue | Numeric field value MUST be greater than or equal to the value. |
| `lt`          | NumericValue | Numeric field value MUST be less than the value.             |
| `lte`         | NumericValue | Numeric field value MUST be less than or equal to the value. |

Regular expressions MUST use
[RE2 syntax](https://github.com/google/re2/wiki/Syntax) for cross-implementation
consistency.

#### Case Insensitivity

If `case_insensitive` is `true`, the match is performed without regard to case.
This applies to all match types including `exact`, `regex`, `starts_with`,
`ends_with`, and `contains`.

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
| `link_trace_id`      | string (hex)        | Link trace ID, authored as hex and matched as bytes (matches if span has link). See [Bytes and Identifier Fields](#bytes-and-identifier-fields). |

See [AttributePath](#attributepath) for path syntax.

##### TraceField Enum Values

| Value                             | Type   | Description                    |
| --------------------------------- | ------ | ------------------------------ |
| `TRACE_FIELD_NAME`                | string | The span name.                 |
| `TRACE_FIELD_TRACE_ID`            | bytes  | The trace identifier. See [Bytes and Identifier Fields](#bytes-and-identifier-fields). |
| `TRACE_FIELD_SPAN_ID`             | bytes  | The span identifier. See [Bytes and Identifier Fields](#bytes-and-identifier-fields). |
| `TRACE_FIELD_PARENT_SPAN_ID`      | bytes  | The parent span identifier. See [Bytes and Identifier Fields](#bytes-and-identifier-fields). |
| `TRACE_FIELD_TRACE_STATE`         | string | The W3C tracestate.            |
| `TRACE_FIELD_RESOURCE_SCHEMA_URL` | string | Resource schema URL.           |
| `TRACE_FIELD_SCOPE_SCHEMA_URL`    | string | Scope schema URL.              |
| `TRACE_FIELD_SCOPE_NAME`          | string | Instrumentation scope name.    |
| `TRACE_FIELD_SCOPE_VERSION`       | string | Instrumentation scope version. |

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

| Type          | Value Type | Description                                                    |
| ------------- | ---------- | -------------------------------------------------------------- |
| `exact`       | string     | String field value MUST equal the specified string exactly.    |
| `regex`       | string     | String field value MUST match the regular expression.          |
| `exists`      | boolean    | If `true`, field MUST exist. If `false`, field MUST NOT exist. |
| `starts_with` | string     | String field value MUST begin with the specified literal string. |
| `ends_with`   | string     | String field value MUST end with the specified literal string. |
| `contains`    | string     | String field value MUST contain the specified literal substring. |
| `equals`      | Value      | Non-string field value MUST equal the typed value. See [Typed and Comparison Matching](#typed-and-comparison-matching). |
| `gt`          | NumericValue | Numeric field value MUST be greater than the value.          |
| `gte`         | NumericValue | Numeric field value MUST be greater than or equal to the value. |
| `lt`          | NumericValue | Numeric field value MUST be less than the value.             |
| `lte`         | NumericValue | Numeric field value MUST be less than or equal to the value. |

Regular expressions MUST use
[RE2 syntax](https://github.com/google/re2/wiki/Syntax) for cross-implementation
consistency.

#### Case Insensitivity

If `case_insensitive` is `true`, the match is performed without regard to case.
This applies to all match types including `exact`, `regex`, `starts_with`,
`ends_with`, and `contains`.

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

Implementations MUST be fail-open: a malformed, invalid, or failing policy MUST
NOT cause telemetry to be lost or incorrectly modified. Errors MUST be isolated
to the offending policy — every other policy continues to operate normally.

Policy errors fall into three categories, distinguished by when they occur:
parse errors, compilation (validation) errors, and runtime evaluation errors.

#### Parse Errors

A parse error occurs when a policy cannot be decoded from its serialized form
(e.g., malformed protobuf, invalid YAML/JSON, or a structurally invalid policy
message).

- A policy that fails to parse MUST be skipped.
- Other policies MUST continue to be parsed and executed.

#### Compilation Errors

After a policy is parsed, implementations validate its contents before using it
to process telemetry. This is the **compilation** stage. Validation is performed
independently per policy.

A policy is invalid if any of the following hold:

| Condition                                                     | Example                                                                            |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| A matcher specifies no field selector.                        | empty `field` oneof                                                                |
| A field selector references an unspecified well-known enum.   | `LOG_FIELD_UNSPECIFIED`, `METRIC_FIELD_UNSPECIFIED`, `SPAN_KIND_UNSPECIFIED`, etc. |
| An attribute selector has an empty path.                      | `log_attribute: []`                                                                |
| A matcher specifies no match condition where one is required. | empty `match` oneof                                                                |
| A `regex` matcher or `redact` pattern is not valid RE2.       | `regex: "([a-z"`                                                                   |
| A `hex_value` is not valid hexadecimal.                       | `hex_value: "xyz"`, odd-length hex                                                 |
| A `keep` value is malformed.                                  | `keep: "banana"`, fractional rate-limit values                                     |
| A trace sampling configuration is invalid.                    | `percentage` out of range, unknown `mode`                                          |
| A transform `rename` has an empty `to` target.                | `rename: { from: ..., to: "" }`                                                    |
| A transform operation references an invalid field selector.   | empty path, unspecified enum, or no field set                                      |

Implementations:

- MUST NOT apply an invalid policy to telemetry. An invalid policy MUST be
  inert: it MUST NOT match, keep, drop, sample, or transform any telemetry. In
  particular, an invalid matcher or transform operation that cannot be compiled
  MUST be skipped in a way that prevents the policy from matching, rather than
  defaulting to a permissive value that could alter telemetry handling.
- MUST NOT abort compilation of the remaining policies in a batch because one
  policy is invalid. Valid policies in the same batch MUST still compile and
  execute.
- SHOULD collect all validation errors for a policy rather than stopping at the
  first, so operators can correct every problem in a single pass.

A compilation error is distinct from a parse error: the policy is well-formed
but semantically invalid. Implementations MAY handle both the same way (skip and
report) but SHOULD distinguish them in diagnostics.

#### Runtime Evaluation Errors

A runtime evaluation error occurs while applying a compiled, valid policy to a
specific telemetry record (e.g., a type mismatch when evaluating a field).

- If a policy fails to evaluate against a record, the telemetry MUST pass
  through unmodified by that policy.
- Runtime evaluation errors MUST NOT cause telemetry loss.

#### Error Reporting

Implementations SHOULD make policy errors observable:

- Compilation errors for a policy MUST be reported via the `errors` field of
  `PolicySyncStatus` (a list of human-readable error strings) so the policy
  server and operators can identify broken policies. A policy with compilation
  errors MUST be reported even when it has no match-hit or match-miss counters.
- Error strings SHOULD identify the location of each problem within the policy,
  for example `log: match[0]: attribute has empty path`,
  `trace: keep: <reason>`, or
  `log: transform: redact[2]: invalid regex "...": <reason>`.
- Implementations SHOULD log policy parse, compilation, and evaluation errors
  for debugging.

### Disabled Policies

Policies with `enabled: false` MUST NOT be evaluated. Implementations MUST treat
disabled policies as if they do not exist.

### Match Tracking

Implementations MUST track match hits and misses for each policy. These counters
are reported via `PolicySyncStatus.match_hits` and
`PolicySyncStatus.match_misses`. Compilation errors for a policy are reported
alongside these counters via `PolicySyncStatus.errors`; see
[Error Handling](#error-handling).

Counters are only incremented for policies whose matchers fire. If a policy's
matchers do not match a telemetry record, neither counter is incremented for
that policy.

A **match hit** is counted when a policy matches a telemetry record and the
record's final keep outcome is consistent with what the policy intended. A
**match miss** is counted when a policy matches a telemetry record but a more
restrictive policy's keep decision overrides the outcome — resulting in the
record being dropped or sampled out against the matching policy's intent.

More precisely, after all matching policies contribute their `keep` values and
the most restrictive value is applied:

- If the record is **kept**, all matching policies record a **hit**.
- If the record is **dropped**, the policy (or policies) whose `keep` value was
  the most restrictive — and therefore caused the drop — record a **hit**. All
  other matching policies record a **miss**.

#### Sampling and Rate Limiting

For probabilistic sampling (`keep: "N%"`) and rate limiting (`keep: "N/s"`,
`keep: "N/m"`, `keep: "N/Ds"`, `keep: "N/Dm"`), the hit/miss outcome depends on
the per-record decision:

- When the sampling or rate limiting decision is **keep**, less restrictive
  matching policies record a **hit** (their intent was not overridden).
- When the sampling or rate limiting decision is **drop**, less restrictive
  matching policies record a **miss** (the record was dropped against their
  intent). The policy that imposed the sampling or rate limit records a **hit**.

#### Example

Given 3 log records and 2 policies:

- `keep-info`: matches `severity_text = "INFO"` → `keep: all`
- `drop-health`: matches body contains `"health"` → `keep: none`

| Record                        | `keep-info`  | `drop-health` | Outcome |
| ----------------------------- | ------------ | ------------- | ------- |
| `"health check ok"` (INFO)    | miss         | hit           | dropped |
| `"user action logged"` (INFO) | hit          | _(no match)_  | kept    |
| `"database error"` (ERROR)    | _(no match)_ | _(no match)_  | kept    |

Result: `keep-info` reports 1 hit / 1 miss. `drop-health` reports 1 hit / 0
misses.

The first record matches both policies, but `drop-health` (`keep: none`) is more
restrictive and causes the drop. `keep-info` records a miss because its intent
(`keep: all`) was overridden. `drop-health` records a hit because its decision
was applied. The third record matches neither policy, so neither counter is
incremented for either policy.

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

Example with nested attribute access and targeted redaction:

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
        regex: '(?i)^(bearer\s+).+$'
        replacement: "$1[REDACTED]"
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

Example with typed equality and numeric comparison:

```yaml
id: drop-successful-request-logs
name: Drop logs for successful HTTP responses
log:
  match:
    # gte + lt match an integer attribute; the literals are ints, not strings
    - log_attribute: ["http.response.status_code"]
      gte: 200
    - log_attribute: ["http.response.status_code"]
      lt: 400
  keep: none
```

```yaml
id: drop-cache-hit-logs
name: Drop logs flagged as cache hits
log:
  match:
    # equals matches the boolean value directly
    - log_attribute: ["cache.hit"]
      equals: true
  keep: none
```

Example matching a bytes identifier field by hex:

```yaml
id: keep-spans-for-trace
name: Keep all spans for one trace
trace:
  match:
    # trace_id is bytes; the hex literal is decoded once and compared as bytes
    - trace_field: TRACE_FIELD_TRACE_ID
      exact: "4bf92f3577b34da6a3ce929d0e0e4736"
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
7. Tracks match hits and misses according to the match tracking semantics.
8. Validates policies per the [compilation error](#compilation-errors) rules,
   keeps invalid policies inert without aborting valid policies in the same
   batch, and reports compilation errors via `PolicySyncStatus.errors`.

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
