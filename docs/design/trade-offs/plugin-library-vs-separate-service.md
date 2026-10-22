# Rules engine: plugin library or separate service

## Context

The project is about learning to build an MCP server, not backend services. The rules engine and the MCP server needed a clear separation of concerns, so the two would not be mixed together.

## Options considered

### Option 1: Plugin-style library inside the server

#### Description

The engine is a library with a plugin-style interface. The MCP server discovers and calls whatever plugins are registered, and it is bundled in the same Lambda deployment.

#### Pros

- Adding a new topic is a self-contained module that does not touch the core server code.
- No network hop between the server and the engine, so no extra infrastructure or cost.
- If the engine is split out later, the boundary is already clean.

#### Cons

- The engine is not independently deployable at first.

### Option 2: Separate deployed service

#### Description

The engine is fully decoupled and runs as its own service that the MCP server calls over a network.

#### Pros

- Fully decoupled from the MCP server from day one.

#### Cons

- More infrastructure, more moving parts, and more cost, which cuts against the keep-it-cheap goal.

## Decision

Start with a library that acts as a plugin, with a clean interface. It can potentially move to its own service in the future.
