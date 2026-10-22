# End-to-end tests: select by environment or by coverage

## Context

The same end-to-end test code is reused for every environment, but not every environment should run every test. The workflow needs a way to pick which tests run.

## Options considered

### Option 1: Parameter based on the environment

#### Description

The script is called with the environment name, and the runner maps dev to every test and prod to a safe subset.

#### Pros

- One end-to-end codebase.
- The mapping lives in the runner, so the workflow steps stay identical.

#### Cons

- The test runner has to know about environments, which the author did not want.
- Adding a new environment in the middle would mean changing the end-to-end script.

### Option 2: Parameter based on coverage

#### Description

The script is called with a coverage level: all or smoke. Tests are tagged smoke or left untagged. An environment variable per GitHub Environment holds the coverage, set to all in dev and smoke in prod.

#### Pros

- Separation of concerns: the runner only knows which tests to run and never learns the environment exists.
- A new environment needs only a new account, a new GitHub Environment with its own variable, and one line in the promotion order, with no end-to-end or workflow code change.
- The full suite can be run against prod by changing one variable.
- The end-to-end project stays testable locally by picking a coverage level.

#### Cons

- Even the smoke subset needs credentials to sign in, so prod needs its own dedicated read-only test user.

## Decision

Coverage-based parameter: all or smoke. The workflow reads the coverage from a per-environment variable.
