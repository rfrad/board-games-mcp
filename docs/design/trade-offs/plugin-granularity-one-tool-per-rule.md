# Plugin granularity: one plugin per rule or plugins with many tools

## Context

The goal was to add a plugin or rule with the least change to the code. Claude, as the future user of the MCP server, also needs to understand clearly where to go and where to start.

## Options considered

### Option 1: One plugin per rule, one tool each

#### Description

Each plugin is one specific rule, with the MCP server exposing one named tool per plugin, such as teach turn sequence or teach core mechanics. Each tool thinly wraps a call into the registry with the plugin ID baked in.

#### Pros

- A clear tool list, so Claude reliably picks the right topic without guessing.

#### Cons

- Adding a plugin also means adding a few lines in the server to register its tool, so it is not zero-change.

### Option 2: Plugins that each expose a list of tools with their own metadata

#### Description

A plugin is a set of rules, for example one for teaching and one for state tracking, and may be linked to a database. It exposes a list of tools, each carrying its own description and metadata. At startup the server loops over every plugin and builds the tool list from that metadata.

#### Pros

- Adding a plugin with its metadata is all that is needed, and nothing changes in the MCP server.
- The MCP server holds no metadata itself.

#### Cons

- The plugin definition has to handle a list of tools instead of a single tool.

## Decision

Plugins list their own tools with embedded metadata, loaded once at startup, and the server never hardcodes a tool name or description.
