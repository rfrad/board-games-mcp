# Lambda integration: Hono or custom translation code

## Context

The MCP library (version 2, see [mcp-sdk-v1-vs-v2.md](mcp-sdk-v1-vs-v2.md)) takes a web-standard `Request` and returns a `Response`. Behind an API Gateway HTTP API, the Lambda receives an event object and must return a response object instead. Something has to translate between the two, for two routes: `/mcp` and the protected resource metadata page.

## Options considered

### Option 1: Hono with its AWS Lambda adapter

#### Description

`handle(app)` from `hono/aws-lambda` translates the Lambda event. `createMcpHonoApp` from `@modelcontextprotocol/hono` creates the app. It needs two settings on Lambda: `allowedHosts: [DOMAIN_NAME]`, and `host: '0.0.0.0'`, which turns off the localhost-only defaults.

#### Pros

- The adapter is maintained and tested by others, and handles HTTP API events, including base64-encoded bodies.
- It is the setup the MCP library documents officially, so there is a recipe to follow.
- JSON body parsing with a size limit is included.
- Easy to add routes later.

#### Cons

- Two extra dependencies: `hono` and `@modelcontextprotocol/hono`.
- Defaults assume a localhost server: without `allowedHosts`, every request gets a 403.
- With the default host, a request carrying a non-localhost `Origin` header is also rejected, so `host: '0.0.0.0'` must be set too.

### Option 2: Custom translation code

#### Description

Roughly 30 to 40 lines that turn the Lambda event into a `Request` and the `Response` back into a Lambda response, with a simple check on the path for the two routes.

#### Pros

- No extra dependencies.
- Fully visible, consistent with choosing the official SDK over a wrapper framework for transparency.

#### Cons

- Edge cases (encoded bodies, headers, query strings) must be handled and unit-tested by the project.

### Option 3: AWS Lambda Web Adapter

#### Description

A Lambda layer that runs a normal web server inside the function.

#### Pros

- None were raised in the discussion.

#### Cons

- An extra, heavier layer than either option above.

## Decision

Hono, with `allowedHosts` set to the domain name and `host` set to `'0.0.0.0'`. The adapter is maintained and tested, and the library documents it officially. The custom code's transparency gain was judged smaller than the risk of getting edge cases wrong.
