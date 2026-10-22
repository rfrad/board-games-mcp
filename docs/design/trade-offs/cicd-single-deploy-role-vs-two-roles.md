# CI/CD: one deploy role or a second narrower role

## Context

The early design had two deploy roles: a broader short-lived one for infrastructure and a narrow one that could only update the Lambda code. The question was whether to keep a second, lighter role for least privilege.

## Options considered

### Option 1: Two deploy roles

#### Description

A second, lighter role is used for application-only deploys, alongside the infrastructure role.

#### Pros

- Least privilege: the pipeline that runs constantly holds almost no power.

#### Cons

- A second workflow, a second role and more manual setup steps.

### Option 2: One deploy role

#### Description

One workflow and one deploy role. The role can only assume the CDK bootstrap roles, where the real power sits. It is already scoped by its trust policy to one repository and one environment, plus the reviewer gate.

#### Pros

- Simpler: one workflow, one role, and fewer manual setup steps.
- Already restricted by the trust policy and the reviewer gate.

#### Cons

- Gives up the extra least-privilege separation of a lighter role.

## Decision

One single workflow and one single deploy role. A separate release pipeline for code-only Lambda deploys is also not built.
