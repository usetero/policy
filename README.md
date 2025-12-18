# Policies

Atomic, portable rules for processing telemetry. From the creator of [Vector](https://vector.dev).

## The problem

Every telemetry tool has the same configuration problem. Datadog Agent, OTel Collector, Vector, Fluent Bit—they all work the same way: you write a config that defines processing rules, and data flows through sequentially.

This breaks down:

- Configs grow into thousands of lines that nobody fully understands
- You need to understand the whole config to safely change any part
- Performance degrades as you add rules—a hundred regex patterns might work, a thousand won't
- AI can't help because the config is too interconnected

We saw this happen across thousands of Vector deployments. Configs that started clean became unmaintainable. Everyone hits the same wall.

## A different model

```yaml
id: drop-checkout-debug-logs
name: Drop checkout debug logs
log:
  match:
    resource.service.name: checkout-api
    severity_text: DEBUG
  keep: none
```

A policy does one thing. You read it and know exactly what it does. No context needed.

## How it works

```
┌─────────────────────────────────────────────────────────────┐
│                      Traditional                            │
│                                                             │
│   Log ──▶ Component A ──▶ Component B ──▶ Component C ──▶ Out  │
│              │               │               │              │
│          bottleneck      bottleneck      bottleneck         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                       Policies                              │
│                                                             │
│                    ┌─▶ Policy 1 ─┐                         │
│                    ├─▶ Policy 2 ─┤                         │
│   Log ──▶ Match ──┼─▶ Policy 3 ─┼──▶ Merge ──▶ Out        │
│                    ├─▶ Policy 4 ─┤                         │
│                    └─▶ Policy N ─┘                         │
│                                                             │
│                 parallel execution                          │
└─────────────────────────────────────────────────────────────┘
```

Telemetry arrives. The runtime matches against all policies in parallel. No sequential bottleneck. Matching policies contribute to fixed stages—`keep` first, then `transform`. If any policy drops the telemetry, it's gone. Otherwise transforms apply and it flows out.

Policies are independent by design. Ten thousand policies execute as fast as ten.

## What this unlocks

**Scale.** Tens of thousands of policies without performance degradation. Drop exactly the log patterns you want, not broad categories.

**AI generation.** Each policy is a bounded problem. No DAG to reason about. AI can generate and manage policies at scale.

**Portability.** Policies use OpenTelemetry's data model. Same policy works across any runtime that implements the spec.

**Simplicity.** Add a policy without understanding the others. Remove one without fear. Each one stands alone.

## Get started

- **[Edge](https://github.com/usetero/edge)** — A minimal, lightweight proxy that enforces policies. Deploy it anywhere in your infrastructure—before your tools, after them, as a sidecar. Takes minutes to set up.

- **[policy-rs](https://github.com/usetero/policy-rs)** — Rust SDK for implementing the policy spec in your own tools.

- **[policy-zig](https://github.com/usetero/policy-zig)** — Zig SDK for implementing the policy spec in your own tools.
