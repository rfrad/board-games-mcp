# Concurrency control and content versioning

## Context

A review of the design flagged two questions: what happens if two requests hit a cold Lambda at the same moment, and whether the DynamoDB table needs versioning when a tool's steps are updated later, for example to fix a typo in a rule.

## Options considered

### Option 1: Add concurrency control and content versioning

#### Description

Guard against different requests seeing different versions of a tool's content while it is being updated.

#### Pros

- Protects against a request reading a tool's steps while the content is being updated.

#### Cons

- The scenario is extremely unlikely with a single content writer and low traffic.

### Option 2: Do nothing

#### Description

Rely on how Lambda already works. Two simultaneous requests normally spin up two separate concurrent executions, with no shared state and no race condition between them.

#### Pros

- Nothing extra to build or maintain.
- Request isolation is free and automatic in Lambda.

#### Cons

- A theoretical window remains where different requests could see different versions mid-update.

## Decision

Explicitly considered and dismissed. No concurrency control or content versioning is added.
