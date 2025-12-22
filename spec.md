# Policy Specification

Status: Draft

## Abstract

This document specifies the structure and semantics of telemetry policies. A policy is an atomic, portable rule for processing telemetry data. Policies are designed for implementation across the telemetry ecosystem using OpenTelemetry's data model.

## Overview

A policy declares a single intent: match specific telemetry and apply an action. Policies are independent units that do not reference each other and do not depend on execution order. This independence enables parallel evaluation and scaling to large policy sets without performance degradation.

Policies use the [OpenTelemetry data model](https://opentelemetry.io/docs/specs/otel/overview/) for field references, ensuring portability across any runtime that implements this specification.

## Design Principles

This specification follows the [OpenTelemetry Specification Principles](https://opentelemetry.io/docs/specs/otel/specification-principles/):

- **User Driven**: Policies address real-world telemetry processing needs.
- **General**: The specification defines behavior, not implementation details.
- **Stable**: Changes preserve backward compatibility.
- **Consistent**: Concepts apply uniformly across telemetry signals.
- **Simple**: The policy model is minimal and unambiguous.

Additionally, policies adhere to these constraints:

- **Atomic**: A policy serves one intent with one matcher and one action set.
- **Self-contained**: A policy MUST NOT reference or depend on other policies.
- **Fail-open**: Policy evaluation failures MUST NOT cause telemetry loss.
- **Idempotent**: Applying the same policy multiple times produces the same result.

## Policy Structure

### Required Fields

A policy MUST contain the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique identifier for the policy. |
| `name` | string | Human-readable name. |

### Optional Fields

A policy MAY contain the following fields:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `description` | string | empty | Explanation of the policy's purpose. |
| `enabled` | boolean | `true` | Whether the policy is active. |
| `created_at_unix_nano` | fixed64 | - | Creation timestamp in Unix epoch nanoseconds. |
| `modified_at_unix_nano` | fixed64 | - | Last modification timestamp in Unix epoch nanoseconds. |
| `labels` | KeyValue[] | empty | Metadata labels for routing and organization. |

### Target

A policy MUST specify exactly one target. The target defines which telemetry signal the policy applies to and contains the matching and action configuration.

Currently defined targets:

| Target | Description |
|--------|-------------|
| `log` | Log record processing. See [Log Target](#log-target). |

Future versions MAY define additional targets (e.g., `metric`, `trace`).

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

A log target MUST contain at least one matcher. A log target MUST specify either a `keep` value other than `"all"` or a `transform`, or both.

### Log Matching

Matchers identify which log records a policy applies to. Multiple matchers are combined with AND logic: all matchers MUST match for the policy to apply.

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

| Selector | Type | Description |
|----------|------|-------------|
| `log_field` | LogField enum | Well-known log record field. |
| `log_attribute` | string | Log record attribute by key. |
| `resource_attribute` | string | Resource attribute by key. |
| `scope_attribute` | string | Instrumentation scope attribute by key. |

##### LogField Enum Values

| Value | Description |
|-------|-------------|
| `LOG_FIELD_BODY` | The log message body. |
| `LOG_FIELD_SEVERITY_TEXT` | Severity as string (e.g., "DEBUG", "INFO", "ERROR"). |
| `LOG_FIELD_TRACE_ID` | Associated trace identifier. |
| `LOG_FIELD_SPAN_ID` | Associated span identifier. |
| `LOG_FIELD_EVENT_NAME` | Event name for event logs. |
| `LOG_FIELD_RESOURCE_SCHEMA_URL` | Resource schema URL. |
| `LOG_FIELD_SCOPE_SCHEMA_URL` | Scope schema URL. |

#### Match Types

A matcher MUST specify exactly one match type:

| Type | Value Type | Description |
|------|------------|-------------|
| `exact` | string | Field value MUST equal the specified string exactly. |
| `regex` | string | Field value MUST match the regular expression. |
| `exists` | boolean | If `true`, field MUST exist. If `false`, field MUST NOT exist. |

Regular expressions MUST use [RE2 syntax](https://github.com/google/re2/wiki/Syntax) for cross-implementation consistency.

#### Negation

If `negate` is `true`, the match result is inverted. A matcher that would match does not match, and vice versa.

### Keep

The `keep` field controls whether matching telemetry survives processing. It unifies dropping, sampling, and rate limiting into a single concept.

| Value | Description |
|-------|-------------|
| `"all"` | Keep all matching telemetry. This is the default. |
| `"none"` | Drop all matching telemetry. |
| `"N%"` | Keep N percent of matching telemetry (0-100). |
| `"N/s"` | Keep at most N records per second. |
| `"N/m"` | Keep at most N records per minute. |

Implementations MUST support `"all"` and `"none"`. Implementations SHOULD support percentage-based sampling. Implementations MAY support rate limiting.

When multiple policies match the same telemetry with different `keep` values, the most restrictive value MUST be applied:

1. `"none"` takes precedence over any other value.
2. Lower percentages take precedence over higher percentages.
3. Rate limits are evaluated independently per policy.

### Log Transform

The `transform` field specifies modifications to apply to log records that survive the keep stage.

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

The field selector uses the same options as LogMatcher: `log_field`, `log_attribute`, `resource_attribute`, or `scope_attribute`.

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

If `upsert` is `false` and the target field already exists, the operation MUST be a no-op. If `upsert` is `true`, the target field MUST be overwritten.

#### LogAdd

Inserts a new field with a specified value.

```
LogAdd {
  field:  <field selector>  // REQUIRED, exactly one
  value:  string            // REQUIRED
  upsert: boolean           // OPTIONAL, defaults to false
}
```

If `upsert` is `false` and the field already exists, the operation MUST be a no-op. If `upsert` is `true`, the field MUST be overwritten.

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

All matching policies contribute their `keep` values. The runtime evaluates these values and applies the most restrictive result. If telemetry is dropped or sampled out, processing stops.

### Stage 2: Transform

All matching policies contribute their transform operations. Operations execute in the defined order: remove → redact → rename → add. Within each operation type, if multiple policies target the same field, the result is implementation-defined but MUST be deterministic.

Implementations MUST process policies in a consistent order (e.g., alphabetically by ID) to ensure reproducible results.

## Runtime Requirements

### Evaluation

Implementations MAY evaluate policies concurrently. The independence of policies enables parallel matching without coordination.

### Error Handling

Implementations MUST be fail-open:

- If a policy fails to parse, it MUST be skipped. Other policies MUST continue to execute.
- If a policy fails to evaluate (e.g., invalid regex at runtime), the telemetry MUST pass through unmodified by that policy.
- Policy failures MUST NOT cause telemetry loss.

Implementations SHOULD log policy evaluation errors for debugging.

### Disabled Policies

Policies with `enabled: false` MUST NOT be evaluated. Implementations MUST treat disabled policies as if they do not exist.

## YAML Representation

While the canonical format is Protocol Buffers, policies MAY be represented in YAML for human authoring. The YAML structure MUST map directly to the protobuf schema.

Example policy in YAML:

```yaml
id: drop-checkout-debug-logs
name: Drop checkout debug logs
description: Remove debug-level logs from checkout-api to reduce volume
enabled: true
log:
  match:
    - resource_attribute: service.name
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
    - resource_attribute: service.name
      exact: payment-api
  transform:
    remove:
      - log_attribute: user.password
    redact:
      - log_attribute: user.email
      - log_attribute: user.phone
        replacement: "[PHONE]"
    add:
      - log_attribute: sanitized
        value: "true"
        upsert: true
```

## Conformance

An implementation conforms to this specification if it:

1. Correctly parses valid policies as defined in this document.
2. Evaluates matchers according to the specified semantics.
3. Applies keep logic with proper precedence rules.
4. Executes transform operations in the specified order.
5. Maintains fail-open behavior for all error conditions.
6. Respects the `enabled` field.

Implementations MAY support a subset of features (e.g., omit rate limiting) but MUST clearly document unsupported features.

## References

- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)
- [OpenTelemetry Log Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [RE2 Regular Expression Syntax](https://github.com/google/re2/wiki/Syntax)
- [RFC 2119 - Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)
- [Policy OTEP](https://github.com/open-telemetry/opentelemetry-specification/pull/4738)
