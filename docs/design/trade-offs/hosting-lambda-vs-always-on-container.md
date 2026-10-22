# Hosting: Lambda or an always-on container

## Context

The project must be as cheap as possible. It is a development project, and paying around £100 a month for a server used only while developing is not affordable.

## Options considered

### Option 1: Lambda, serverless

#### Description

A Lambda function runs the MCP server behind API Gateway.

#### Pros

- Scales to zero, so it costs essentially nothing when not in use.

#### Cons

- None were raised in the discussion.

### Option 2: Container on ECS or App Runner

#### Description

A small container, closer to a traditional always-on server.

#### Pros

- None were raised in the discussion.

#### Cons

- Always-on cost, which the project cannot afford.
- More setup involved, such as building a Docker container.

## Decision

Lambda, because the whole point is to save costs and serverless is the most reasonable way to do that. It also keeps the implementation small, since a container needs more setup.
