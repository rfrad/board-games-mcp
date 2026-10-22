# CI/CD: reviewer gate on prod only or on both environments

## Context

The diff step prints what a deploy will change before anything is deployed. A required reviewer on a GitHub Environment pauses the workflow until the change is approved or rejected. The question was which environments should have that gate.

## Options considered

### Option 1: Required reviewer on prod only

#### Description

Prod is the only environment where the gate pauses. The diff still prints in dev, but dev passes straight through.

#### Pros

- Dev deploys happen on their own.

#### Cons

- The diff in dev cannot be read before it is deployed.

### Option 2: Required reviewer on both dev and prod

#### Description

The gate is a GitHub Environment setting, so ticking required reviewers on dev as well needs no workflow change.

#### Pros

- The diff can be read in dev before approving it.

#### Cons

- Every pull request pauses at the dev gate until approved, so a pull request deploy does not happen on its own.

## Decision

Both environments require a reviewer. It can be unticked on dev later if it becomes tedious.
