# Observability: where to send logs

## Context

Every error is logged as structured JSON. The server needs observability. We needed a destination that could be searched by field, without a recurring cost.

## Options considered

### Option 1: Datadog

#### Description

Send logs to Datadog from the start.

#### Pros

- Stronger filtering, dashboards and plotting of log data than CloudWatch Logs Insights.

#### Cons

- No meaningful free tier for logs. The free plan excludes log ingestion, and only a 14-day trial covers it.
- Ingestion and indexing are charged, so it is not free for a project that runs indefinitely.

### Option 2: CloudWatch Logs

#### Description

Use the AWS-native log destination, which Lambda already writes to.

#### Pros

- Free at low volume.
- Logs Insights discovers fields automatically in structured JSON logs, so filtering on fields such as the tool ID or level needs no manual parsing.
- No extra service to add, which keeps the project simple.

#### Cons

- Field search is less polished than Datadog's.

## Decision

Use CloudWatch Logs for now. Datadog is deferred and can be added later without changing the logging format.
