# Error handling: HTTP status codes and error detail

## Context

Two kinds of failure exist: the database being unavailable, which looks like a server error, and a step or rule that does not exist, which looks like a user error. Claude needs a way to tell them apart and continue the conversation.

## Options considered

### Option 1: Real HTTP error codes

#### Description

Return a 500 for a server failure and a 400 for a user error, as in normal HTTP.

#### Pros

- Follows the usual HTTP convention for server and user errors.

#### Cons

- MCP has two error channels, and HTTP status codes only reflect whether the transport worked. A protocol-level error is one Claude cannot recover from conversationally.

### Option 2: HTTP 200 with an error flag in the tool result

#### Description

Any backend problem returns 200. The error is carried inside the response as a tool result with the error flag set, plus a message. Every error logs five fields: error description with stack, friendly user message, machine-readable error code, developer explanation, and a runbook link.

#### Pros

- Claude can read the message and recover gracefully, instead of the exchange breaking.
- The five logged fields separate what the end user sees from what a developer needs.

#### Cons

- Roles have to be managed in Cognito, because detail depends on a claim in the token.

## Decision

Return 200 for everything from the MCP. Log all five fields every time. If the bearer token carries the developer role, return everything. Otherwise return only the user-facing message.
