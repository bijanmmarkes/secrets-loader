Automatically load secrets into environment variables at runtime when running with [Bref](https://bref.sh).

Supports two AWS backends:

- **AWS Secrets Manager** — store all env vars as a single JSON secret, loaded in **1 API call** per cold start
- **AWS SSM Parameter Store** — load individual parameters via the `bref-ssm:` prefix (existing behavior)

This package is separate so that its dependencies are not installed for all Bref users. Install it only if you need runtime secret loading.

## Installation

```
composer require bref/secrets-loader
```

## Usage

Read the full Bref documentation: https://bref.sh/docs/environment/variables.html#secrets

### Secrets Manager

Create a JSON secret per environment:

```bash
aws secretsmanager create-secret \
  --name api/production \
  --secret-string '{"DB_PASSWORD":"secret123","API_KEY":"abc-xyz"}'
```

**Global import** — all JSON keys become env vars with 1 API call:

```yaml
provider:
    environment:
        BREF_SECRETS_MANAGER: api/${sls:stage}
    iam:
        role:
            statements:
                - Effect: Allow
                  Action: secretsmanager:GetSecretValue
                  Resource: arn:aws:secretsmanager:${aws:region}:${aws:accountId}:secret:api/*
```

**Individual import** — load specific secrets per env var:

```yaml
provider:
    environment:
        # Plain string secret
        VENDOR_KEY: bref-secret:vendor/api-key
        # Extract a specific key from a JSON secret
        SPECIFIC_VAR: bref-secret:api/${sls:stage}:SPECIFIC_KEY
```

Multiple references to the same secret name are deduplicated into a single API call.

### SSM Parameter Store

```yaml
provider:
    environment:
        MY_PARAMETER: bref-ssm:/my-app/my-parameter
```

`MY_PARAMETER` is replaced at runtime with the value stored at `/my-app/my-parameter` in SSM.

### Combining both backends

You can use all three sources together. Priority order (highest wins):

1. `bref-ssm:` — explicit SSM per-variable mapping
2. `bref-secret:` — explicit Secrets Manager per-variable mapping
3. `BREF_SECRETS_MANAGER` — bulk Secrets Manager import

```yaml
provider:
    environment:
        BREF_SECRETS_MANAGER: api/${sls:stage}
        VENDOR_KEY: bref-secret:vendor/api-key
        LEGACY_PARAM: bref-ssm:/old/ssm/parameter
```

### Why Secrets Manager?

| | SSM Parameter Store | Secrets Manager |
|---|---|---|
| API calls per cold start | ceil(N/10) | **1** |
| TPS limit | 1,000 | **10,000** |
| Max value size | 4 KB | **64 KB** |

