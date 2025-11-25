# Policy Spec

This repository houses an experimental version of the specification for the basis of the OpenTelemetry (Otel) Policy. The Otel policy is proposed in [this OTEP](https://github.com/open-telemetry/opentelemetry-specification/pull/4738). The Tero Policy specification is designed to be merged in the future with the OpenTelemetry Policy specification. The goal of this repository is to explore what the Policy specification should have. This methodology is inspired by that from the goals set forth by Lightstep in developing OTAP [[1](https://github.com/open-telemetry/otel-arrow/), [2](https://github.com/open-telemetry/community/issues/1332), [3](https://opentelemetry.io/blog/2023/otel-arrow/)].

# Goals

1. Policies are generic to telemetry type and data format
2. Policies are identified by a unique type
3. Policies must be idempotent
4. Policies are implementation agnostic

# Constraints

1. This repository does not define a set of policies, only the proto format and expected behavior therein
2. This repository does not implement a policy provider nor an underlying transport
3. Policies must be mergable and define a strategy for handling overlapping clauses for the same type
