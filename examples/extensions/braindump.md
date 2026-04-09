# Custom Policies

## Extension Design

Principles:

1. Agents MUST specify which custom policies it accepts
2. Custom policies SHOULD

## Handling S3 exporter initialization

The question is if we should initialize S3 in the policy or if we should
initilaize S3 externally and reference it from the policy.

Were we to initialize in the policy, we could dynamically change the S3 location
and characteristics from the control plane

### Pros of Policy Initialization

1. Dynamic S3 location
2. Dynamic partitioning
3. Encapsulated configuration

### Cons of Policy Initialization

1. Increases custom policy size
2. Requires setting an id for lookups, could result in collision
3. Note: is this a limitation? (see below)

#### Collision on initialization

Say policy A sets config like so:

```json
{
  "id": "dump-s3",
  "name": "dump waste to S3",
  "custom": {
    "type": "tero-s3-dump",
    "match": [
      { "log_field": "body", "regex": "[error|failed].*$" },
      { "resource_attribute": ["service.name"], "regex": "checkout-api" }
    ],
    "keep": ".01%",
    "destination": {
      "id": "eu-bucket",
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
```

Were policy B to change the prefix, we would need to re-initialize the config,
but then you also get into an ordering problem. I.e. policy A and B disagree
about which came first, and there would be no good way to reconcile
determanistically. We could hash the config and then use that to key into a map
of initialized S3 exporters. This is a better key to the map than trying to deal
with manual ids. Are there issues with this? We should probably constrain the
implementation to allow a maximum amount of destinations. I do not think it
makes sense to allow the configuration of things like batching, format, etc. for
this config, at least not initially, it can be additive.

### Pros of Implementation Initialization

1. Smaller policy payload
2. Initialize client on startup
3. Client controls scaling characteristics
4. Easier policy failure (if id does not exist, fail load)

### Cons of Implementation Initialization

1. Cannot add or change destination
2. Less reactive if there are problems

## Decision

I _think_ initially we should do the policy initialization. It allows for better
UX and customer experience. We could allow users to setup integrations in the
UI, and then specify the configuration in the policy. We could also have them
set a proper serviceaccount for use by the edge implementation.

## Routing

Now the question is what data to route? Do we route everything or just what is
matched? Is this policy an extension of the log/metric/trace matching ones?
Although the initial use case is to dump "artifacts", I could imagine for
compliance someone will also want to use this same logic to s3 dump what is
sampled. How may we modify the existing policies? Maybe! the custom policy
references? No... that removes the no references guarantee.

Maybe the right way to do this would be having custom be a top level field, not
part of the target. That way we don't have to duplicate the config. It would
then be up to the implementation to use the custom config if its present... how
would this look?

```json
{
  "id": "keep-error-logs",
  "name": "keep-error-logs",
  "log": {
    "match": [
      { "log_field": "body", "regex": "[error|failed].*$" },
      { "resource_attribute": ["service.name"], "regex": "checkout-api" }
    ],
    "keep": ".01%"
  },
  "custom": {
    "type": "tero-s3-dump",
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
}
```

It's a bit verbose and we'd need to make it so that target can be nothing or
minimal. The config part in there is ultimately just gonna be raw bytes.
Serialization may be a bit odd, but i think that's the best way to do it. That
being said, we'll need to keep an index reference for any custom config which
makes this a bit more gross, but maybe not bad.

I was worried about a case when you want a custom target, but then I considered
that we're always matching on telemetry in the same way. That way our "custom"
stuff here should really just be considered an "extension".

In zig we'll keep either an index or a reference to the linked custom config is
available. We can actually modularize and load them based on a static variable
for type.

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
  "extension": {
    "type": "tero-s3-dump",
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
}
```

So this feels right overall. The extension pattern should allow us to add in new
things over time and if we properly split out the modules we can just add them
in to the distro as we like. Maybe there's a way we can do that in the collector
too.

## Multiple extensions?

Should we allow for multiple extensions? Is there a world where someone would
want multiple extensions? The way i'm currently designing the dump extension
should allow me to dump to file too. I imagine it's possible that someone would
want to dump to disk and s3?
