# API Gateway type: HTTP API or REST API

## Context

The type of API Gateway was not decided, and it changes the deploy permissions and how Cognito is integrated. The question was which is cheapest and most secure.

## Options considered

### Option 1: HTTP API

#### Description

An HTTP API with a Cognito JWT authorizer.

#### Pros

- Cheaper: $1.00 per million requests, against $3.50 for REST.
- Validates JWT tokens against the configured issuer and audience, rejecting unauthorized requests before they reach the backend.
- Deploys automatically, so there is no separate redeploy step.
- Simpler JWT authorizer for Cognito.

#### Cons

- Does not support AWS WAF, per-consumer usage plans or IP-based resource policies.

### Option 2: REST API

#### Description

A REST API, which needs an explicit redeploy.

#### Pros

- Supports AWS WAF, usage plans and throttling, and IP-based resource policies.

#### Cons

- More expensive per request.
- Needs an explicit redeploy.
- WAF would add a monthly cost that works against the scale-to-zero approach.

## Decision

HTTP API with a Cognito JWT authorizer plus throttling. The missing WAF matters little because access is invite-only and every call needs a valid token. Abuse is covered by route-level throttling, a reserved concurrency limit on the function, and a budget alert on each account.
