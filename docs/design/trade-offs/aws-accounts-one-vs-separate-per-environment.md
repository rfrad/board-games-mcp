# AWS accounts: one for both environments or one each

## Context

The CI/CD design needed to decide how the dev and prod environments are separated in AWS.

## Options considered

### Option 1: One AWS account, separate stacks

#### Description

Dev and prod live in the same account as separate stacks.

#### Pros

- Cheaper and simpler for a project this size.

#### Cons

- Less isolation between the environments.

### Option 2: Two accounts

#### Description

Dev and prod are separate member accounts under the main AWS Organization, with identical CDK stacks. Secrets are stored per GitHub Environment.

#### Pros

- Cleaner isolation between the environments.

#### Cons

- Each account needs a one-time CDK bootstrap, a manually created OIDC provider and a deploy role.
- The teaching content has to be loaded into each account's DynamoDB table by hand.

## Decision

Two separate member accounts, with the exact same structure.
