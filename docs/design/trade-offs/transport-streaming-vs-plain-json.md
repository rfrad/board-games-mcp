# Transport: streaming or plain JSON

## Context

Streamable HTTP lets the server choose per request between a plain JSON response and a stream. The tools return teaching content instantly, with no long-running work.

## Options considered

### Option 1: Streaming

#### Description

The server opens a stream to send progress updates or server-initiated messages while a tool is working.

#### Pros

- Lets a long-running tool show progress along the way.

#### Cons

- Streaming through Lambda behind API Gateway has historically been awkward, because API Gateway tends to buffer responses.
- Not needed here, since no tool does slow work.

### Option 2: Plain JSON, stateless

#### Description

The server replies with a plain JSON response and runs with no session tracking.

#### Pros

- Avoids the API Gateway buffering problem.
- Fits tools that return content instantly.
- Even a large block of lore is still one request and one response, so plain JSON handles it.

#### Cons

- No progress updates for slow tools, if any are added later.

## Decision

Plain JSON responses and a stateless server for v1. Streaming would be revisited only if a tool needed it, and it is a transport setting plus possibly an infrastructure change, not a change of SDK.
