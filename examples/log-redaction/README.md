# Log Redaction

Examples for targeted replacement inside log bodies.

## Body Replacements

- `redact-api-keys-in-log-body.yaml` replaces every fixed-length API key token
  matched in the log body. The replacement regex has no capture groups.
- `redact-password-query-param.yaml` replaces a password value while preserving
  both captured delimiters around it.

```yaml
id: redact-api-keys-in-log-body
name: Redact API keys in log body

log:
  match:
    - log_field: LOG_FIELD_BODY
      regex: "api_key="
  transform:
    redact:
      - log_field: LOG_FIELD_BODY
        regex: 'api_key=[A-Za-z0-9]{32}'
        replacement: "api_key=[REDACTED]"
```

```yaml
id: redact-password-query-param
name: Redact password query parameter in log body

log:
  match:
    - log_field: LOG_FIELD_BODY
      regex: "password="
  transform:
    redact:
      - log_field: LOG_FIELD_BODY
        regex: '([?&]password=)[^&\s]+(&session_id=)'
        replacement: "$1[REDACTED]$2"
```
