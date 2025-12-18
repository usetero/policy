# PCI Compliance

PCI-DSS requires you to protect cardholder data. Auditors don't just want to know that you're doing it—they want to see exactly how. "We redact PAN in our log pipeline" isn't good enough. They'll ask where, how, and whether it covers all the cases.

With policies, the answer is simple: point to the files.

## Two Layers

**Foundation policies** are blanket rules that apply everywhere. If a log has a CVV field, redact it. Period. These are your baseline guarantees.

**Generated policies** handle specific log event types. The control plane identifies which log events contain sensitive fields and generates redaction policies for each. Different events have different fields—`card_number` vs `billing_address` vs `payment_method`. Each gets its own policy.

## The Audit Story

Auditor: "How do you ensure CVV is never stored in logs?"

You: `cat foundation/redact-cvv.yaml`

```yaml
id: redact-cvv
name: Redact CVV codes

log:
  match:
    - log_attribute: cvv
      exists: true
  transform:
    redact:
      - log_attribute: cvv
```

That's the entire answer. One file. No grep through pipeline configs. No "I think it's somewhere in the VRL transform."

## The Policies

### Foundation (human-written)

- `redact-pan.yaml` - Redact primary account numbers wherever they appear
- `redact-cvv.yaml` - Redact CVV codes wherever they appear
- `drop-payment-debug.yaml` - No debug logs from payment services (defense in depth)

```yaml
id: drop-payment-debug
name: Drop debug logs from payment services

log:
  match:
    - resource_attribute: service.name
      regex: "^payment-"
    - log_field: severity_text
      exact: DEBUG
  keep: none
```

### Generated (per log event)

**payment-api/**
- `redact-transaction-logged.yaml` - Redact card_number, account_id
- `redact-payment-processed.yaml` - Redact card_last_four, billing_address

```yaml
id: payment-api-redact-payment-processed
name: Redact payment processed events

log:
  match:
    - resource_attribute: service.name
      exact: payment-api
    - log_field: body
      regex: "payment processed"
  transform:
    redact:
      - log_attribute: card_last_four
      - log_attribute: billing_address
```

**checkout-api/**
- `redact-order-submitted.yaml` - Redact payment_method, billing_email

## Why This Works

**Auditability.** Each policy is a discrete, auditable control. You can review them, version them, get sign-off on them.

**Completeness.** The control plane identifies every log event type that contains sensitive fields. No manual discovery. No gaps.

**Separation of concerns.** Security team owns the foundation policies. Service teams own their generated policies. Clear ownership, clear accountability.
