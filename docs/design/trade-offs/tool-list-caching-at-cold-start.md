# Tool list: caching at cold start

## Context

On a Lambda cold start, the startup code calls the list of tools on every plugin, which queries DynamoDB for each tool's metadata. The step content is deliberately excluded, so only the menu is loaded.

## Options considered

### Option 1: Cache the tool list in memory

#### Description

The tool list is loaded once at cold start and kept in memory for as long as that Lambda instance stays warm.

#### Pros

- Only the first request after a cold start pays the DynamoDB query. Later requests on the same warm instance skip it.
- Avoids repeated work, latency and DynamoDB cost, since the list barely changes.
- The memory cost is negligible: only metadata, a few hundred kilobytes at most even with a few hundred tools.

#### Cons

- The first request after a cold start is slower, by a few hundred milliseconds. This was judged fine for a V1.

### Option 2: Query DynamoDB on every request

#### Description

Re-read the tool metadata from DynamoDB for each request.

#### Pros

- None were raised in the discussion.

#### Cons

- Repeated work for the same answer every time, adding latency and a little DynamoDB cost for no benefit.

## Decision

Cache the tool list in memory. Step content is fetched one step at a time per request and never cached in bulk, so the two do not compete for memory.
