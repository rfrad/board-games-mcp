# Authentication: token check in API Gateway or in the Lambda

## Context

This follows from choosing an HTTP API (see [api-gateway-http-vs-rest.md](api-gateway-http-vs-rest.md)). When a request has no valid token, Claude expects a 401 whose WWW-Authenticate header points to the protected resource metadata page. The HTTP API's JWT authorizer returns a 401 without that pointer, and no way to add one was found. A REST API may allow customising this 401, but that was not checked. Without the pointer, Claude falls back to probing /.well-known/oauth-protected-resource on the server's domain.

## Options considered

### Option 1: Keep the token check in API Gateway

#### Description

The HTTP API's Cognito JWT authorizer validates every token before the Lambda runs. The metadata route has no authorizer. Claude finds that route through its fallback, since the 401 carries no pointer.

#### Pros

- Matches the existing design, with no extra code.
- Requests without a valid token never reach the Lambda.

#### Cons

- Relies on Claude's fallback, which Claude's documentation describes as a fallback rather than the normal path.
- Follows the MCP specification less closely, which may matter if other clients are supported later.

### Option 2: Validate the token in the Lambda

#### Description

The route has no authorizer. The Lambda validates each token itself with a JWT library and returns a 401 with the resource_metadata pointer.

#### Pros

- Returns exactly the 401 the MCP specification describes.

#### Cons

- More code to write and maintain, in a security-sensitive area.
- Requests without a valid token reach and invoke the Lambda.

## Decision

Keep the token check in API Gateway and rely on Claude's fallback. Claude is the only target client for v1, and this keeps the design simple. Revisit if the fallback stops working or other clients are supported.
