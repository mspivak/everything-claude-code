---
name: localstack
description: Set up and manage LocalStack for local AWS emulation in the Ezra monorepo (S3, SQS, Secrets Manager). Use when asked to add, configure, or debug LocalStack for local development.
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

# LocalStack Setup for Ezra Monorepo

LocalStack emulates AWS services locally for development.

**Services to emulate** (exclude ECS — Docker handles that):
- **S3** — data room and bundle storage
- **SQS** — `bundle-processing-{env}` queue + DLQ
- **Secrets Manager** — API keys and credentials

## Arguments

- `$ARGUMENTS`: target directory (defaults to `/Users/mauro/Developer/ezra/mono`)

## Step 1 — Read Existing State

Before making changes, read:
- `docker-compose.yml` (or `compose.yaml`) in the target repo root
- `.env.example` and `.env` files for S3/AWS config
- `temporal-worker/terraform/modules/sqs/` or similar for exact queue names
- `temporal-worker/lambda/bundle-processor/handler.py` for what secrets/queues are consumed

## Step 2 — Add LocalStack to docker-compose

Add a `localstack` service to the existing `docker-compose.yml`. Do NOT create a separate file — integrate into the existing compose.

```yaml
localstack:
  image: localstack/localstack:4
  container_name: localstack
  ports:
    - "4566:4566"
  environment:
    - SERVICES=s3,sqs,secretsmanager
    - DEFAULT_REGION=us-east-1
    - EAGER_SERVICE_LOADING=1
    - LOCALSTACK_HOST=localstack
  volumes:
    - "./scripts/localstack:/etc/localstack/init/ready.d"
    - localstack_data:/var/lib/localstack
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:4566/_localstack/health"]
    interval: 5s
    timeout: 3s
    retries: 10
```

Add `localstack_data:` to the top-level `volumes:` block.

## Step 3 — Write the Init Script

Create `scripts/localstack/01-init.sh` — this runs automatically when LocalStack is ready.

```bash
#!/usr/bin/env bash
set -euo pipefail

AWS_CMD="aws --endpoint-url=http://localhost:4566 --region us-east-1"
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1

# S3
$AWS_CMD s3 mb s3://ezra-data-rooms-dev

# SQS — DLQ first, then main queue with redrive policy
DLQ_URL=$($AWS_CMD sqs create-queue --queue-name bundle-processing-dlq-dev --query QueueUrl --output text)
DLQ_ARN=$($AWS_CMD sqs get-queue-attributes --queue-url "$DLQ_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text)

$AWS_CMD sqs create-queue \
  --queue-name bundle-processing-dev \
  --attributes "{\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"$DLQ_ARN\\\",\\\"maxReceiveCount\\\":\\\"3\\\"}\"}"

# Secrets Manager — dummy values for local dev
$AWS_CMD secretsmanager create-secret --name ezra/local/temporal-api-key --secret-string "local-temporal-key"
$AWS_CMD secretsmanager create-secret --name ezra/local/google-genai-api-key --secret-string "${GOOGLE_GENAI_API_KEY:-dummy}"
$AWS_CMD secretsmanager create-secret --name ezra/local/openai-api-key --secret-string "${OPENAI_API_KEY:-dummy}"
$AWS_CMD secretsmanager create-secret --name ezra/local/ezra-api-key --secret-string "test-secret-api-key"

echo "LocalStack init complete"
```

Make the script executable: `chmod +x scripts/localstack/01-init.sh`

## Step 4 — Update Environment Variables

In `.env.example` and `.env`, ensure these values are set:

```
S3_ENDPOINT_URL=http://localhost:4566
S3_ACCESS_KEY_ID=test
S3_SECRET_ACCESS_KEY=test
S3_REGION=us-east-1
AWS_ACCESS_KEY_ID=test
AWS_SECRET_ACCESS_KEY=test
AWS_DEFAULT_REGION=us-east-1
```

## Step 5 — Verify

After applying changes, run:

```bash
docker compose up localstack -d
docker compose logs localstack --tail=30
```

Confirm LocalStack is healthy:
```bash
curl -s http://localhost:4566/_localstack/health | python3 -m json.tool
```

Verify resources were created:
```bash
AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test aws \
  --endpoint-url=http://localhost:4566 --region us-east-1 \
  s3 ls

AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test aws \
  --endpoint-url=http://localhost:4566 --region us-east-1 \
  sqs list-queues
```

## Key Facts

- LocalStack endpoint: `http://localhost:4566` (all services, single port)
- Credentials: always `test` / `test` locally
- Region: `us-east-1` (LocalStack default)
- Init scripts in `ready.d/` run in alphanumeric order after LocalStack passes health check
- `EAGER_SERVICE_LOADING=1` prevents cold-start delays on first API call
- Data persists between restarts via the named volume `localstack_data`
- `LOCALSTACK_HOST=localstack` ensures container-internal DNS resolves correctly for presigned URLs

## Troubleshooting

**Init script did not run**: Check `docker compose logs localstack` for `[init] running script`. Ensure the script has execute permission (`chmod +x`).

**Presigned URLs broken**: Set `LOCALSTACK_HOST=localstack` in the LocalStack env so generated URLs use the container hostname. Clients outside Docker need to override to `localhost:4566`.

**Secret not found at runtime**: The lambda/worker reads secrets by ARN. Locally, construct the ARN as `arn:aws:secretsmanager:us-east-1:000000000000:secret:<name>`. Set `AWS_ENDPOINT_URL=http://localhost:4566` in the service's env so boto3 routes to LocalStack.
