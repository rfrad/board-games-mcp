# CI/CD: how the pipelines are structured

## Context

The design needed a deployment pipeline for dev and prod. The author's first idea was one pipeline for the infrastructure and one for the application, because the infrastructure changes a few times while the application changes a lot.

## Options considered

### Option 1: Separate infrastructure and application pipelines

#### Description

The infrastructure pipeline uses a short-lived, broader role. The application pipeline uses a longer-lived, narrow role that can only update the Lambda code. A pull request is either for infrastructure or for application, never both.

#### Pros

- Smaller blast radius: the pipeline that runs constantly holds almost no power.
- Day-to-day pushes are fast, since updating Lambda code takes seconds while a full CDK deploy takes minutes.

#### Cons

- A lot of machinery for a small project: two pipelines, two roles and a pull request check.
- Ordering has to be settled when both change, and CDK and the live Lambda code can drift apart.
- An agent file only guides the AI, so a check would be needed to make the one-concern-per-PR rule enforceable.
- Four workflows in total, with dev and prod each repeating steps that must be kept in step.

### Option 2: Dev then prod chained in one reusable workflow

#### Description

One reusable deploy workflow takes the environment as an input. Dev deploys and is tested first, then prod calls the same workflow.

#### Pros

- Dev and prod run the same steps, so they cannot drift apart.

#### Cons

- Prod depends on dev being successful, which the author did not want.

### Option 3: One workflow with a matrix over environments

#### Description

A merge to main runs the checks, then fans out into one identical run per environment, in parallel and independent: diff, accept gate, deploy, then end-to-end tests.

#### Pros

- Exactly the same steps for dev and prod, with no doubt that they deploy the same way.
- Dev failing never blocks prod, or the other way round, with fail-fast set to false.
- Adding a new environment is one new matrix entry.
- The prod run stays pending until approved, at zero cost.

#### Cons

- Prod is no longer gated on dev being green, so the dev run has to be checked before approving.
- GitHub expires an unapproved run after 30 days.

## Decision

Option 3: one reusable workflow and a matrix over environments, with no dependency between dev and prod. Triggers are a pull request build, a merge to main, and a release build. If an approval expires, a new hotfix release version can be cut.
