# Authentication: Cognito or Auth0

## Context

Authentication is built in from day one, so that access can be given to individual users rather than through a shared secret. Auth0 has been used in previous projects.

## Options considered

### Option 1: Amazon Cognito

#### Description

Cognito acts as the OAuth provider. API Gateway validates the tokens before requests reach the Lambda.

#### Pros

- Native to AWS, so API Gateway, Lambda and Cognito form one coherent stack and keep the whole project in the AWS stack.
- Free tier of 10,000 monthly active users, so a handful of users costs nothing.

#### Cons

- Auth0 is arguably nicer to use.

### Option 2: Auth0

#### Description

A third-party authentication provider, already used in other personal projects.

#### Pros

- Already familiar from earlier projects.
- Arguably nicer to use than Cognito.

#### Cons

- Brings in a third-party vendor, which slightly dilutes building the whole project on AWS.

## Decision

Cognito. It is not chosen because it is better, but because it keeps the whole stack AWS-native.
