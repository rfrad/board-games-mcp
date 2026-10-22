# Cognito IDs: stack outputs or stored variables

## Context

The end-to-end tests need the Cognito user pool ID and client ID to sign in. The question was where those identifiers should come from.

## Options considered

### Option 1: Store them as GitHub secrets or variables

#### Description

The identifiers are saved in GitHub as secrets or variables and passed to the tests.

#### Pros

- None were raised in the discussion.

#### Cons

- One more set of values to store and keep in step with the deployed stack.

### Option 2: Read them from CDK stack outputs

#### Description

CDK exposes them as stack outputs. The deploy workflow reads them after the deploy and passes them as parameters to the end-to-end tests. They are not stored anywhere.

#### Pros

- Nothing extra to store.
- Low risk even though they appear in the public deploy log: they are identifiers, not credentials. Sign-up is closed and invite-only, and Cognito locks out repeated failed logins.

#### Cons

- The identifiers appear in the public deploy log.

## Decision

Not stored anywhere. They come from the stack outputs after each deploy.
