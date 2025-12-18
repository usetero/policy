# Cost Control

You're spending too much on logs. Most of them are noise—debug statements, health checks, verbose INFO logs that nobody looks at. The problem is identifying which logs are waste and which are valuable. Pipeline configs make this hard because the logic is tangled together. You can't see what's being dropped without tracing through transforms and conditionals.

Policies make it obvious. Each file is one decision. You can see exactly what's being dropped and why.

## Two Layers

**Foundation policies** are human-written blanket rules. They apply broadly based on severity, patterns, or resource attributes. Things like "drop all debug logs" or "sample health checks." These are the baseline.

**Generated policies** are created by the control plane for each distinct log event type. The system fingerprints your logs by their message patterns, understands which are valuable and which are waste, and generates a policy for each. You might have hundreds of these—one per log event type across all your services.

## Before and After

Here's what dropping debug logs and health checks looks like in Vector VRL:

```toml
[transforms.filter_logs]
type = "remap"
inputs = ["source"]
source = '''
  if .level == "debug" {
    abort
  }
  if match!(.message, r'GET /health') {
    abort
  }
'''
```

And in OpenTelemetry Collector:

```yaml
processors:
  filter:
    logs:
      exclude:
        match_type: regexp
        bodies:
          - "GET /health"
        severity_texts:
          - "DEBUG"
```

With policies:

```yaml
# foundation/drop-debug-logs.yaml
id: drop-debug-logs
name: Drop all debug logs

log:
  match:
    severity_text: DEBUG
  keep: none
```

```yaml
# foundation/drop-health-checks.yaml
id: drop-health-checks
name: Drop health check logs

log:
  match:
    body: /GET \/health/
  keep: none
```

Same result. But now each decision is a separate file. You can enable or disable them independently. Different teams can own different policies. You can audit exactly what's being filtered.

## The Policies

### Foundation (human-written)

- `drop-debug-logs.yaml` - Drop all DEBUG severity logs
- `drop-health-checks.yaml` - Drop health check request logs
- `sample-info-logs.yaml` - Keep only 10% of INFO logs

### Generated (per log event)

The control plane analyzes your telemetry and generates policies for each distinct log event type:

**checkout-api/**
- `drop-cart-item-added.yaml` - High volume, low value cart events
- `drop-request-received.yaml` - Redundant request logging
- `sample-order-validated.yaml` - Keep 50%, strip verbose attributes

**user-service/**
- `drop-session-heartbeat.yaml` - Constant noise from session keepalives
- `drop-cache-hit.yaml` - Verbose cache logging

## The Result

These 8 policies target the highest-volume, lowest-value logs. Each one is obvious—you can read it and understand exactly what it does. No pipeline context required. No graph to trace. Just files that say what they do.
