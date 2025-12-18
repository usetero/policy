# Cost Control

You're spending too much on logs. Most of them are noise—debug statements, health checks, verbose INFO logs that nobody looks at. 

The problem isn't knowing what to filter. It's managing it at scale. You have hundreds of services, thousands of distinct log event types. Pipeline configs don't scale to this. They become unmaintainable walls of conditionals that nobody wants to touch.

## Two Layers

**Foundation policies** are human-written blanket rules. Drop all debug logs. Sample health checks. These apply broadly and rarely change.

**Generated policies** are created by the control plane for each distinct log event type. The system fingerprints your logs by their message patterns, understands which are valuable and which are waste, and generates a policy for each. You might have hundreds of these—one per log event type across all your services.

## Why This Works

**Scale.** You can't manage 500 rules in a pipeline config. You can manage 500 policy files. Each one is independent. No shared state. No ordering dependencies.

**Ownership.** Platform team owns foundation policies. Service teams own their generated policies. No merge conflicts in a shared config file.

**Visibility.** "What are we dropping from checkout-api?" → `ls generated/checkout-api/`. That's it.

**AI generation.** The control plane analyzes your telemetry and generates policies automatically. You can't generate pipeline configs—they require global reasoning. You can generate atomic policies.

## The Policies

### Foundation (human-written)

- `drop-debug-logs.yaml` - Drop DEBUG and TRACE severity logs
- `drop-health-checks.yaml` - Drop health check request logs

```yaml
id: drop-debug-logs
name: Drop debug and trace logs

log:
  match:
    - log_field: severity_text
      regex: "^(DEBUG|TRACE)$"
  keep: none
```

### Generated (per log event)

**checkout-api/**
- `drop-cart-item-added.yaml` - High volume, low value cart events
- `drop-request-received.yaml` - Redundant request logging
- `sample-order-validated.yaml` - Keep 50%, strip verbose attributes

```yaml
id: checkout-api-sample-order-validated
name: Sample order validated logs

log:
  match:
    - resource_attribute: service.name
      exact: checkout-api
    - log_field: body
      regex: "order validated"
  keep: 50%
  transform:
    remove:
      - log_attribute: cart_contents
      - log_attribute: validation_trace
```

**user-service/**
- `drop-session-heartbeat.yaml` - Constant noise from session keepalives
- `drop-cache-hit.yaml` - Verbose cache logging

## The Result

Eight policies. Each one is obvious—you can read it and understand exactly what it does. No pipeline context required. No graph to trace. Add more services, add more policies. The complexity stays flat.
