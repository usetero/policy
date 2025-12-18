# Policy Specification

This document defines the policy specification—a format for expressing telemetry processing rules that are atomic, portable, and executable at scale.

The spec is designed for implementation across the telemetry ecosystem. Policies use OpenTelemetry's data model, making them portable across any runtime that implements this specification: collectors, agents, proxies, or SDKs.

## Motivation

Telemetry tools share a configuration problem. Whether you're using the OpenTelemetry Collector, Vector, Fluent Bit, or a vendor agent like Datadog's, the pattern is the same: you write a configuration that defines processing rules, and data flows through sequentially.

This model breaks down in predictable ways:

- **Configs grow organically.** You find a problem, add a rule. Another problem, another rule. A year later you have thousands of lines that nobody fully understands. The configuration becomes the accumulated knowledge of every issue you've ever hit, but that knowledge isn't legible. You can't look at line 847 and know why it exists or what depends on it.

- **Configs require global reasoning.** To safely change one part, you need to understand the whole. Data flows through a DAG of components—what shape is it at this point? What came before? What breaks if you modify this? The cognitive load grows with the config size until changes become risky and reviews become superficial.

- **Configs don't scale.** You want to drop a hundred noisy log patterns? That might work. A thousand? Ten thousand? Performance degrades. The architecture wasn't designed for this level of specificity. So you compromise—drop broad categories and accept losing signal with the noise.

- **AI can't help.** Ask a model to modify a 5,000 line configuration with complex interdependencies and watch it break things. The context is too large, the dependencies too hidden. The format wasn't designed for machine generation.

These problems compound. Teams stop making improvements because changes are risky. Costs grow because nobody can safely add the filtering rules that would reduce them. Knowledge stays trapped in the config, inaccessible to anyone who wasn't there when each rule was added.

Policies are a different model. Instead of a monolithic configuration, you have independent rules—each one atomic, self-contained, and understandable in isolation. The system is designed from the ground up to handle tens of thousands of policies without degradation. The format is simple enough that AI can generate and manage policies at scale.

This specification defines that model.

## Explanation

### What Policies Are

A policy is a single rule that matches telemetry and declares what should happen to it. One policy, one intent.

```yaml
id: drop-checkout-debug-logs
name: Drop checkout debug logs
log:
  match:
    resource.service.name: checkout-api
    severity_text: DEBUG
  keep: none
```

```yaml
id: redact-payment-pii
name: Redact PII from payment service
log:
  match:
    resource.service.name: payment-api
  transform:
    redact:
      - attributes.user.email
      - attributes.user.phone
```

The first policy drops debug logs from checkout-api. The second redacts PII from payment-api logs. You can read each one and understand exactly what it does without any other context. They don't depend on each other. They don't require understanding a pipeline or DAG. Each is complete in itself.

A system might have ten policies or ten thousand. Each one remains this simple. The complexity of "how do all these policies execute together" is handled by the runtime, not by the policy author.

### Core Constraints

Policies are designed around a set of constraints that enable scale and simplicity:

- **Atomic.** A policy serves one intent. It has one matcher and declares what should happen to matching telemetry—whether to keep it, transform it, or both.

- **Self-contained.** A policy doesn't reference other policies. It doesn't depend on execution order. You can read it in isolation and know what it does.

- **Independent execution.** When multiple policies match the same telemetry, they all execute. There's no explicit ordering between policies—the runtime handles coordination through fixed stages.

- **Portable.** Policies use OpenTelemetry's data model for matching. The same policy works across any runtime that implements this spec: collectors, agents, proxies, SDKs.

These constraints are features, not limitations. They're what enable a system to scale to tens of thousands of policies without becoming unmaintainable.

### Policy Structure

Every policy has the following fields:

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique identifier for the policy. Used for tracking, updates, and metrics. |
| `name` | No | Human-readable name. |
| `description` | No | Explanation of what the policy does and why it exists. |
| `enabled` | No | Whether the policy is active. Defaults to `true`. |
| `log` | One of | Target for log policies. Contains `match`, `keep`, and `transform`. |
| `metric` | One of | Target for metric policies. Contains `match`, `keep`, and `transform`. |

A policy must specify exactly one target: `log` or `metric`. Within the target, `match` is required, and at least one of `keep` (with a value other than `all`) or `transform` must be specified.

The target contains:
- `match` — Predicate that determines which telemetry this policy applies to.
- `keep` — Whether to keep or discard matching telemetry. Defaults to `all`.
- `transform` — Transformations to apply to matching telemetry.

## Internal Details

### Policy Format

Policies are authored in YAML. The format is intentionally minimal—every field has a clear purpose, and there's no room for ambiguity.

A complete policy looks like this:

```yaml
id: sanitize-payment-logs
name: Sanitize payment logs
description: Remove password, redact email, and mark as sanitized
enabled: true
log:
  match:
    resource.service.name: payment-api
  transform:
    remove:
      - attributes.user.password
    redact:
      - attributes.user.email
    add:
      - field: attributes.sanitized
        value: "true"
```

The structure makes the target type explicit (`log` or `metric`). Within `transform`, substages run in a fixed sequence: remove → redact → rename → add.

Field values in `match` can be:

- **Exact strings** — `severity_text: ERROR` matches only "ERROR"
- **Regular expressions** — Wrapped in slashes: `/^5\d{2}$/` matches 500-599
- **Existence checks** — `exists` matches any field that is present (regardless of value)

All match conditions are ANDed together. Every condition must match for the policy to apply.

### Matching

Policies match against the OpenTelemetry data model. For logs, the matchable fields are:

| Field | Description |
|-------|-------------|
| `resource.*` | Resource attributes identifying the source (e.g., `resource.service.name`, `resource.deployment.environment`) |
| `severity_text` | Log severity as a string: TRACE, DEBUG, INFO, WARN, ERROR, FATAL |
| `severity_number` | Log severity as a number (1-24) |
| `body` | The log message body |
| `attributes.*` | Log record attributes (e.g., `attributes.user.id`, `attributes.http.status_code`) |

Field names use dot notation to reference nested structures. This maps directly to OpenTelemetry semantic conventions—`resource.service.name` corresponds to the `service.name` resource attribute in OTel's data model.

To negate a match, use the `not` prefix:

```yaml
log:
  match:
    resource.service.name: checkout-api
    not:severity_text: DEBUG  # matches anything except DEBUG
```

The `not:` prefix inverts the match—the condition succeeds when the field does *not* match the value.

For metrics, the matchable fields are:

| Field | Description |
|-------|-------------|
| `resource.*` | Resource attributes identifying the source |
| `name` | The metric name |
| `attributes.*` | Metric data point attributes (dimensions/tags) |

### Keep

The `keep` field controls whether matching telemetry survives. It unifies dropping, sampling, and rate limiting into a single concept: what percentage or amount of matching telemetry continues to the next stage?

| Value | Meaning |
|-------|---------|
| `all` | Keep everything (default, can be omitted) |
| `none` | Drop everything |
| `N%` | Keep N percent (0-100) |
| `N/s` | Keep at most N per second |
| `N/m` | Keep at most N per minute |

Examples:

```yaml
# Drop all debug logs
id: drop-debug
log:
  match:
    severity_text: DEBUG
  keep: none
```

```yaml
# Sample 10% of healthcheck logs
id: sample-healthchecks
log:
  match:
    body: /healthcheck/
  keep: 10%
```

```yaml
# Rate limit noisy service to 100 logs per second
id: ratelimit-noisy-service
log:
  match:
    resource.service.name: noisy-api
  keep: 100/s
```

If multiple policies match the same telemetry and specify different `keep` values, the most restrictive wins. `none` beats `10%` beats `100%`.

### Transform

The `transform` field specifies modifications to apply to telemetry that survives the `keep` stage. Transformations are grouped by type, and types execute in a fixed order:

```
remove → redact → rename → add
```

This ordering is deliberate:

- **Remove first.** Delete unwanted fields before any other processing.
- **Redact second.** Mask sensitive values in their original field names. This way, a policy targeting `user.email` works regardless of whether another policy later renames that field.
- **Rename third.** Move fields around after they're cleaned. Policies that redact don't need to know about renamed field names.
- **Add last.** Insert new fields into the final structure, after all other modifications.

Each substage operates on the result of the previous one. This ordering eliminates conflicts and makes policies easier to write—you target fields as they exist in the source data, not as they might be transformed by other policies.

```yaml
log:
  match:
    resource.service.name: payment-api
  transform:
    remove:
      - attributes.user.password
      - attributes.user.ssn
    redact:
      - attributes.user.email
      - attributes.user.phone
    rename:
      - from: attributes.user_id
        to: attributes.user.id
    add:
      - field: attributes.sanitized
        value: "true"
```

Within each substage, if multiple policies target the same field, last write wins. This is rare in practice—it usually indicates a user error, and the result is deterministic regardless.

### Stages

When telemetry arrives, the runtime evaluates all policies that match it. Execution proceeds through fixed stages:

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

**1. Keep**

All matching policies contribute their `keep` values. The most restrictive value wins. If telemetry is dropped or sampled out, processing stops.

**2. Transform**

All matching policies contribute their transformations. Substages execute in order: remove → redact → rename → add. Within each substage, all matching policies' actions apply.

### Transform Reference

Each transform type and its syntax:

**remove**

A list of fields to delete:

```yaml
log:
  match:
    resource.service.name: payment-api
  transform:
    remove:
      - attributes.user.password
      - attributes.user.ssn
      - body  # removes the entire log body
```

**redact**

A list of fields to mask. Optionally specify a custom replacement:

```yaml
log:
  match:
    resource.service.name: payment-api
  transform:
    redact:
      - attributes.user.email
      - field: attributes.user.phone
        replacement: "[PHONE REDACTED]"  # optional, defaults to [REDACTED]
```

**rename**

A list of field renames:

```yaml
log:
  match:
    resource.service.name: payment-api
  transform:
    rename:
      - from: attributes.user_id
        to: attributes.user.id
      - from: attributes.ts
        to: attributes.timestamp
```

**add**

A list of fields to insert:

```yaml
log:
  match:
    resource.service.name: payment-api
  transform:
    add:
      - field: attributes.processed_by
        value: policy-engine
      - field: attributes.environment
        value: production
```

### Runtime Behavior

A few rules govern how runtimes execute policies:

- **Fail-open.** If a policy fails to evaluate (malformed regex, missing field, etc.), telemetry passes through unmodified. Policies never cause data loss due to errors.

- **Matching is parallel.** The runtime may evaluate all policies against incoming telemetry concurrently. This is possible because policies are independent.

- **Stage order is fixed.** Runtimes must execute stages in the defined order: Keep → Transform (Remove → Redact → Rename → Add).

- **Most restrictive keep wins.** If multiple policies specify different `keep` values for the same telemetry, apply the most restrictive. `none` beats any percentage, lower percentages beat higher ones.

- **Last write wins.** If multiple policies in the same transform substage target the same field, the result is the last write. Runtimes should process policies in a deterministic order (e.g., alphabetical by ID) so results are reproducible.

### Data Types

This specification covers logs and metrics. The same model applies to both—match on fields, execute actions in stages.

**Logs**

Log policies match against the OTel log data model: resource attributes, severity, body, and log record attributes. All `keep` values and transforms are applicable.

**Metrics**

Metric policies match against resource attributes, metric name, and data point attributes. All `keep` values and transforms are applicable, except:

- `body` does not exist for metrics. Transforms targeting `body` are invalid.

For metrics, `keep` with a percentage samples metric data points. Rate limiting (`N/s`, `N/m`) limits the number of data points per time window.

## Trade-offs and Mitigations

This specification makes deliberate trade-offs in favor of simplicity and scale.

**No user-defined ordering.** You cannot specify that policy A runs before policy B. This is intentional—ordering creates dependencies, and dependencies break the independence that makes policies scale. The trade-off is less flexibility. If you need strict ordering, you need separate processing stages outside the policy system.

**No conditional logic.** Policies don't support if/else or branching. Each policy is a simple predicate and action. Complex conditional logic belongs in your application code, not your telemetry processing. This keeps policies easy to understand and easy to generate.

**No cross-policy references.** A policy cannot reference another policy's output or depend on another policy having run. This limits composition but ensures every policy is self-contained. You can reason about each policy in isolation.

**Most restrictive keep wins.** When multiple policies match the same telemetry, the strictest `keep` value applies. This means a single `keep: none` policy can override others. The alternative—requiring explicit conflict resolution—would add complexity and ordering concerns. The current behavior is predictable: if any matching policy wants to drop telemetry, it's dropped.

**Limited transform types.** The spec defines a small set of transforms: remove, redact, rename, add. It doesn't support arbitrary transformations like regex replacement, field splitting, or computed values. This keeps the execution model simple and fast. Complex transformations belong in application code or a dedicated ETL layer.

These constraints exist because the primary goal is scale—tens of thousands of policies executing efficiently. Every feature that adds complexity makes that goal harder. The spec intentionally stays minimal.

## Prior Art and Alternatives

**Pipeline configurations.** Vector, Fluent Bit, Logstash, and the OTel Collector all use pipeline-based configurations. Data flows through a DAG of components, each transforming the data. This model is flexible but doesn't scale to thousands of rules and requires global reasoning.

**OPA (Open Policy Agent).** OPA provides a general-purpose policy language (Rego) for authorization and admission control. It's powerful but complex—Rego has a learning curve, and policies can have arbitrary logic. This spec is narrower: telemetry processing only, with a minimal set of operations.

**Datadog processing pipelines.** Datadog provides a UI for building log processing rules. Each rule has a matcher and action, similar to policies. The limitation is that rules are vendor-specific and not portable.

**OpenTelemetry Collector processors.** The OTel Collector has processors like `filter`, `attributes`, and `transform` that modify telemetry. These are configured in YAML as part of the pipeline. The spec differs by making each rule independent and portable across runtimes.

This specification draws from all of these but prioritizes independence, portability, and scale over flexibility.

## Open Questions

**Regex syntax.** The spec uses `/pattern/` for regular expressions but doesn't specify the regex dialect. Should this be RE2 (safe, no backtracking), PCRE, or something else? Implementations need to agree for portability.

**Rate limiting scope.** When `keep: 100/s` is specified, what's the scope? Per-policy? Per-matcher? Global across the runtime? This affects how rate limiting behaves in distributed deployments.

**Metric aggregation.** The spec supports dropping and sampling metrics, but not aggregation (e.g., converting histograms to summaries, reducing cardinality by dropping dimensions). Is this a transform type or out of scope?

**Trace support.** This version covers logs and metrics. Traces have different semantics—spans are related, sampling decisions should be consistent across a trace. How should policies apply to traces?

**Negation syntax.** The spec uses `not:field` for negation. Alternatives include a separate `not:` block or a `negate: true` flag on individual matchers. The current approach is concise but may conflict with field names in edge cases.

## Future Possibilities

**Traces.** Extend the spec to support span matching and trace-aware sampling. This requires careful design around trace context and parent/child relationships.

**Richer transforms.** Add transforms like `truncate` (limit string length), `hash` (one-way hash for pseudonymization), or `extract` (parse structured data from strings). Each would need to fit the staged execution model.

**Policy inheritance.** Allow policies to inherit from base policies, reducing duplication. This would need to preserve the self-contained property—the resolved policy should be understandable without chasing references.

**Ecosystem adoption.** Work with OTel Collector, Vector, and other tools to implement the spec natively. Contribute SDKs that make implementation straightforward. The goal is policies as a portable standard, not a single-vendor feature.
