# Language: TypeScript everywhere or a Python engine

## Context

The plan was to build in TypeScript. The idea came up of writing the rules engine in Python and the MCP server in TypeScript, which would reinforce the separation of concerns.

## Options considered

### Option 1: TypeScript for both the server and the engine

#### Description

Everything is written in TypeScript. The plugin-based engine lives as a module inside the same Lambda as the MCP server.

#### Pros

- No network hop and no extra cost.
- Still cleanly separated by the plugin interface, so splitting it out later stays easy.
- The Streamable HTTP transport was, at the time of the guides found, available out of the box in the TypeScript SDK.

#### Cons

- Gives up the stricter separation that two languages would enforce.

### Option 2: Python engine with a TypeScript server

#### Description

The rules engine is written in Python and the MCP server in TypeScript.

#### Pros

- Reinforces the separation of concerns.

#### Cons

- They could no longer be a library import. The engine would have to run as its own API that the TypeScript Lambda calls over a network.
- More infrastructure, more moving parts, and more cost.

## Decision

Keep both in TypeScript for v1, with the plugin-style engine bundled in the same Lambda.
