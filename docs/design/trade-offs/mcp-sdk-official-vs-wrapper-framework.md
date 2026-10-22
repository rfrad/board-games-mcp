# MCP library: official SDK or a wrapper framework

## Context

An SDK handles the protocol work: parsing JSON-RPC messages, managing request IDs and the Streamable HTTP transport. The open question was whether to use the official package directly or a higher-level framework built on top of it.

## Options considered

### Option 1: Official SDK

#### Description

Use the official `@modelcontextprotocol/sdk` package directly, with its server class and Streamable HTTP transport.

#### Pros

- More transparent about what is actually happening.
- Supports stateless mode and a plain JSON response setting, which match the design.
- In stateless mode a new transport and server are created per request, which fits Lambda's isolated invocations.

#### Cons

- Lacks the conveniences a wrapper framework adds.

### Option 2: Wrapper framework

#### Description

A higher-level framework in the style of FastMCP, built on top of the raw SDK, which came up when researching ChatGPT compatibility.

#### Pros

- Adds conveniences on top of the raw SDK.

#### Cons

- Less transparent about what is actually happening.

## Decision

The official SDK, in stateless mode with JSON responses enabled. Streaming could be added later by changing a transport setting, without replacing the SDK.
