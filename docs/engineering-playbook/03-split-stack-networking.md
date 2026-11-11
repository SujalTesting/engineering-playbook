# Module 3: Web System Architecture & Networking

This module explains the architectural boundaries, request lifecycles, and
networking contracts that govern monolithic and split-stack web applications.
The same principles apply whether the edge is Netlify, a CDN, an API gateway,
or a cloud load balancer, and whether the backend runs on a VM, container, or
serverless function.

## 1. Architectural evolution: monolith versus split stack

### Monolithic architecture

In a monolith, frontend rendering, backend business logic, and database access
are packaged and deployed as one application, commonly behind one origin.

Typical characteristics:

- simple relative routing such as `/api/projects`;
- shared host, scheme, and port;
- straightforward cookie scope and same-origin requests;
- one deployment unit and one primary runtime boundary;
- failures and logs are often centralized.

The simplicity is valuable, but a monolith can create coupled releases, larger
blast radius, slower scaling of independent workloads, and less isolated
ownership of frontend and backend changes.

### Split-stack or decoupled architecture

In a split stack, the frontend is hosted separately on a static edge or CDN,
authentication may be delegated to an identity provider such as Supabase, and
API routes may run on serverless functions such as Google Cloud Functions.
Storage, queues, and external services form additional boundaries.

Typical characteristics:

- multiple origins and independently deployed components;
- explicit API, identity, and serialization contracts;
- separate frontend and backend configuration;
- decentralized state and distinct failure domains;
- independent scaling, caching, and release lifecycles.

Decoupling is not automatically better. It introduces CORS, token, timeout,
versioning, observability, and deployment coordination concerns. Choose the
boundary because it provides a clear ownership or scaling benefit, not because
it is fashionable.

### Architecture comparison

| Concern | Monolith | Split stack |
|---|---|---|
| Routing | Mostly internal and relative | Edge, proxy, and service routes |
| Authentication | Often same-origin session cookie | Cookie or explicit token contract |
| Deployment | One coordinated unit | Independent releases and compatibility |
| Scaling | Scale the application unit | Scale frontend, API, and workers separately |
| Main risks | Coupling and blast radius | Network, contract, and configuration drift |
| Debugging | Centralized logs often sufficient | Correlation across multiple services required |

## 2. Request lifecycle and routing

Document both the successful path and at least one failure path. A common
split-stack request looks like this:

```text
Browser
  -> DNS and TLS
  -> CDN or edge host
  -> reverse proxy route (/api/*)
  -> backend function or service
  -> database, queue, or external dependency
  -> JSON response with correlation ID
```

At each hop define:

- protocol, host, port, and route;
- timeout and cancellation behavior;
- authentication and authorization requirements;
- request and response size limits;
- cacheability and invalidation;
- retry ownership;
- correlation and tracing headers;
- expected error translation.

### Same-Origin Policy

The browser's Same-Origin Policy (SOP) restricts scripts from reading
resources on a different origin. An origin is the combination of scheme, host,
and port; `https://app.example.com` and `https://api.example.com` are
different origins even though they share a parent domain.

SOP does not prevent all cross-origin network activity. It primarily controls
what browser scripts can read or mutate, and is supplemented by CORS, cookie
rules, CSP, and other browser security controls.

### Reverse proxies

A reverse proxy or edge route can present a unified public origin:

```text
https://app.example.com/api/projects
  -> edge /api/* route
  -> https://functions.example.net/projects
```

The proxy can simplify browser routing, centralize TLS, apply headers and rate
limits, and avoid browser CORS for proxied calls. It does not remove the need
for backend authorization, and it does not help requests that bypass the proxy
and call the backend origin directly.

Verify proxy behavior for:

- path-prefix stripping or preservation;
- trailing slash normalization;
- query strings and encoded characters;
- HTTP methods and request bodies;
- forwarded host and protocol headers;
- status codes and response headers;
- websocket or streaming support;
- cache and timeout defaults.

### CORS

Cross-Origin Resource Sharing is an HTTP-header contract that allows a server
to declare which origins, methods, and headers a browser may use. A correct
configuration should specify trusted origins rather than using a permissive
wildcard with credentials.

For credentialed requests, confirm all of the following:

- the exact `Origin` is allowed;
- `Access-Control-Allow-Credentials: true` is returned;
- allowed methods and headers include the request;
- preflight `OPTIONS` requests succeed;
- caches vary on `Origin` where required;
- errors also include the necessary CORS headers.

Preflight failures often look like application failures in the browser, so
inspect the network panel and server logs rather than only the JavaScript
exception.

## 3. Authentication and authorization mechanics

Authentication establishes identity. Authorization determines whether that
identity may perform an action on a resource. Every protected backend route
must enforce authorization independently; hiding a button in the frontend is
not an authorization control.

### Cookie-based sessions

The server creates a session and sends a cookie with attributes such as:

- `HttpOnly`, preventing ordinary JavaScript reads;
- `Secure`, requiring HTTPS;
- `SameSite`, limiting cross-site sending;
- narrow `Domain` and `Path` scope;
- an appropriate expiration or inactivity policy.

Cookies reduce direct token exposure to client JavaScript, but they do not
eliminate XSS risk: injected script may still cause authenticated actions in
the user's browser. Protect state-changing operations against CSRF using
SameSite policy, CSRF tokens, origin checks, or an equivalent defense.

### Token-based authentication

JWT or bearer tokens are sent explicitly:

```http
Authorization: Bearer <short-lived-access-token>
```

Tokens are flexible for split-stack services, but storing long-lived tokens in
`localStorage` makes them readable by injected JavaScript. Prefer short-lived
access tokens, secure refresh flows, strict CSP, minimal scopes, and an
approved storage strategy. Never log access tokens or put them in URLs.

Validate issuer, audience, signature, expiration, and required claims on the
server. Use server-side role and resource checks instead of trusting claims
without verifying their integrity and meaning.

### Identity-provider boundaries

When an external provider issues identity tokens, document who validates the
token, how roles map to application permissions, how logout and revocation
work, and what happens when the provider is unavailable. Test expired,
malformed, wrong-audience, revoked, and insufficient-role credentials.

## 4. Backend API contracts and serialization

Distributed systems depend on strict contracts between callers and responders.
Define an API schema using OpenAPI, JSON Schema, generated types, or an
equivalent reviewed artifact.

Each route should specify:

- method and canonical path;
- request headers, parameters, and body schema;
- success response schema;
- status codes and error schema;
- authentication and authorization rules;
- pagination, filtering, and sorting behavior;
- idempotency and retry behavior;
- maximum payload and field lengths;
- deprecation and version-compatibility policy.

Use explicit content types and validate untrusted input at the service
boundary. Responses should not change shape based on an undocumented
environment or failure path.

### Status code baseline

- `200 OK`: successful read or action with a response body.
- `201 Created`: resource created, ideally with its canonical location or ID.
- `202 Accepted`: work accepted for asynchronous processing.
- `204 No Content`: successful action with no response body.
- `400 Bad Request`: malformed or invalid request.
- `401 Unauthorized`: missing or invalid authentication.
- `403 Forbidden`: authenticated but not permitted.
- `404 Not Found`: resource or route is unavailable to the caller.
- `409 Conflict`: state or uniqueness conflict.
- `429 Too Many Requests`: rate limit or capacity policy.
- `5xx`: server or dependency failure; do not expose internal details.

Use a stable machine-readable error code alongside a safe human-readable
message. Clients should branch on the code and status, not fragile prose.

## 5. The HTML error-page and JSON parsing trap

A common split-stack failure occurs when a client expects JSON but receives an
HTML error page:

```text
Client calls /api/departments/
  -> proxy route does not match trailing-slash form
  -> edge returns an HTML 404 document
  -> response.json() throws "Unexpected token '<'"
```

The `<` is often the first character of `<!doctype html>`. The parser error is
only the symptom; the root cause may be a wrong base URL, missing proxy route,
trailing slash mismatch, redirect, authentication page, or deployment drift.

### Defensive client behavior

Before parsing, verify status and content type:

```ts
const response = await fetch(url, requestInit);
const contentType = response.headers.get('content-type') ?? '';

if (!response.ok) {
  throw new ApiError(response.status, await response.text());
}
if (!contentType.includes('application/json')) {
  throw new ApiError(502, `Expected JSON, received ${contentType || 'unknown'}`);
}

const payload = await response.json();
```

The client should preserve the status, request ID, sanitized URL, and a bounded
response preview for diagnostics. Never silently parse an HTML fallback or
return an empty object that makes the test appear successful.

### Contract tests

Add a smoke or contract test that verifies each environment's canonical API
base URL, representative route, content type, status behavior, and error
shape. Test both `/api/resource` and the deliberately unsupported
`/api/resource/` form when trailing-slash behavior is a known risk.

## 6. Resilience and failure behavior

Every outbound dependency needs bounded connect, response, and overall
operation timeouts. Retries must be limited, use backoff and jitter, and apply
only to operations that are safe to repeat. A retry of a non-idempotent create
operation can duplicate data unless the request has an idempotency key.

Use circuit breaking, rate limiting, queueing, or load shedding when a
dependency is unavailable. Propagate cancellation so a disconnected browser
request does not continue consuming server resources. Translate dependency
failures into stable public errors while retaining the internal cause in
protected logs.

## 7. Caching, headers, and edge behavior

Define caching separately for static assets, public reads, authenticated reads,
and mutations. Never cache personalized or sensitive responses without an
explicit, reviewed policy. Use cache-control directives, validators such as
ETag where useful, and invalidation rules tied to deployment or data changes.

Security and operational headers should be intentional, including:

- Content Security Policy;
- strict transport security;
- frame and content-type protections;
- referrer policy;
- request and response correlation headers.

Check that the edge does not strip authentication, tracing, CORS, or cache
headers that the backend contract depends on.

## 8. Environment and deployment parity

Keep local, test, staging, and production configuration structurally similar.
Externalize endpoints, credentials, feature flags, and routing settings, but
validate required values at startup. Fail clearly for an unknown environment;
never silently fall back to a development endpoint.

For each environment, document:

- frontend origin and API origin;
- proxy and DNS routes;
- identity-provider configuration;
- allowed CORS origins;
- database and queue endpoints;
- timeout, retry, and rate-limit policy;
- TLS certificates and secret ownership;
- health and readiness checks.

Use service emulators or containers when they provide meaningful contract
parity. Verify deployment artifacts and routes against the candidate build,
not only against local configuration.

## 9. Observability across boundaries

Generate or accept a correlation ID at the edge and propagate it through API
calls, database spans, queues, and external requests. Record structured logs
with service, route, status, duration, environment, and dependency metadata.

Use distributed traces to answer:

- which hop was slow or failed;
- whether the request reached the backend;
- which dependency produced the error;
- whether a retry or redirect occurred;
- which deployed version handled the request.

Never log passwords, tokens, connection strings, or unredacted personal data.

## 10. Networking diagnosis checklist

When a request fails, inspect in order:

1. Browser URL, method, and resolved origin.
2. DNS and TLS result.
3. Proxy route, path rewriting, and redirects.
4. CORS preflight and response headers.
5. Authentication presence, expiry, and audience.
6. Backend status, content type, and error body.
7. Correlation ID across edge, service, and dependency logs.
8. Database, queue, or external-service health.

This sequence prevents treating a parser error, CORS message, or generic 500 as
the root cause without examining the network contract.

## 11. Architecture review checklist

- [ ] Every service boundary has an owner and documented contract.
- [ ] Canonical routes and trailing-slash behavior are defined.
- [ ] Proxy rewrites preserve methods, query strings, and required headers.
- [ ] Direct and proxied API calls have intentional CORS behavior.
- [ ] Authentication and resource authorization are enforced server-side.
- [ ] Cookies or tokens use a reviewed storage and expiration strategy.
- [ ] Error responses are stable JSON with safe machine-readable codes.
- [ ] Timeout, retry, cancellation, and idempotency policies are documented.
- [ ] Environment routes and identity settings are validated at startup.
- [ ] Correlation IDs and traces cross every network boundary.
- [ ] Health checks, rollback, and deployment verification are available.
