# MCP library: version 1 or version 2

## Context

The design named the official `@modelcontextprotocol/sdk` package, which is version 1. Version 2, published as `@modelcontextprotocol/server`, became the stable release on 27 July 2026. Version 1 now lives on a separate branch and receives bug and security fixes for at least six months after that date. The question was which version to build on.

## Options considered

### Option 1: Version 1

#### Description

`@modelcontextprotocol/sdk` 1.x (1.32.2 at the time of writing), using its Streamable HTTP transport in stateless mode with JSON responses enabled.

#### Pros

- More examples online.
- AI assistants are likely to know it best from their training.

#### Cons

- Legacy line: fixes are only guaranteed until roughly late January 2027.
- Speaks the 2025 protocol versions. Support for the 2026-07-28 revision was not confirmed.

### Option 2: Version 2

#### Description

`@modelcontextprotocol/server` 2.x (2.4.0 at the time of writing). `createMcpHandler(factory)` builds a fresh server for every request, and `responseMode: 'json'` returns plain JSON with no streaming.

#### Pros

- Current stable line.
- The per-request model is exactly the stateless design.
- Serves 2025-era clients and the 2026-07-28 revision from the same handler.
- Works with web-standard `Request` and `Response`, which simplifies the Lambda integration.

#### Cons

- Newer, so fewer examples online.
- AI assistants may default to version 1 habits, so the docs must name the version 2 package and link to its documentation.

## Decision

Version 2. Starting a new project on a line whose support has a fixed end date makes little sense, and version 2's per-request model matches the stateless design directly.
