# Extensions

This example shows how to attach implementation-specific behavior to a standard
policy using the top-level `extensions` array.

Core policy semantics stay the same (`match`, `keep`, `transform`). Extensions
add optional behavior (for example, exporting matched telemetry) without
changing how matching and keep/drop decisions work.

## What This Example Demonstrates

- A normal log policy that matches checkout errors and samples at `0.01%`
- A single extension entry with a namespaced `type`
- Extension schema versioning via `version`
- Extension-specific payload in `config`
- Extension configuration for an S3 destination

## The Policy

- `extension.json` - Match error logs from `checkout-api` and attach an
  `com.usetero/s3-dump` extension

```json
{
  "id": "dump-s3",
  "name": "dump waste to S3",
  "log": {
    "match": [
      { "log_field": "body", "regex": "[error|failed].*$" },
      { "resource_attribute": ["service.name"], "regex": "checkout-api" }
    ],
    "keep": ".01%"
  },
  "extensions": [
    {
      "type": "com.usetero/s3-dump",
      "version": "1.0.0",
      "config": {
        "destination": {
          "kind": "s3",
          "config": {
            "region": "eu-central-1",
            "s3_bucket": "databucket",
            "s3_prefix": "metric",
            "s3_partition_format": "%Y/%m/%d/%H/%M"
          }
        }
      }
    }
  ]
}
```

## Notes

- Extensions are implementation-specific. Runtimes should explicitly document
  which `type` and `version` values they support.
- Extension execution should be non-blocking and off the hot path.
- The original design notes for this example live in `braindump.md`.
