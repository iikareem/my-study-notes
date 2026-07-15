---
tags:
  - api-gateway
  - aws
  - certification
---

**Hub:** [[AWS MOC]] · **Role:** Guide
**Also:** [[API Gateway Abstraction]]

# Amazon API Gateway — Complete Knowledge Guide

---

## Table of Contents

1. [What is Amazon API Gateway?](#1-what-is-amazon-api-gateway)
2. [Where It Fits in Modern Architecture](#2-where-it-fits-in-modern-architecture)
3. [API Types — REST vs HTTP vs WebSocket](#3-api-types--rest-vs-http-vs-websocket)
4. [Core Concepts Deep Dive](#4-core-concepts-deep-dive)
5. [Endpoint Types](#5-endpoint-types)
6. [Integration Types](#6-integration-types)
7. [The Request Lifecycle](#7-the-request-lifecycle)
8. [Request and Response Transformation (VTL)](#8-request-and-response-transformation-vtl)
9. [Authorization and Security Model](#9-authorization-and-security-model)
10. [Stages, Deployments, and Versioning](#10-stages-deployments-and-versioning)
11. [Usage Plans, API Keys, and Throttling](#11-usage-plans-api-keys-and-throttling)
12. [Caching](#12-caching)
13. [CORS](#13-cors)
14. [Logging, Monitoring, and Observability](#14-logging-monitoring-and-observability)
15. [Custom Domain Names and TLS](#15-custom-domain-names-and-tls)
16. [WebSocket APIs In Depth](#16-websocket-apis-in-depth)
17. [HTTP APIs (v2) In Depth](#17-http-apis-v2-in-depth)
18. [VPC Integration and Private APIs](#18-vpc-integration-and-private-apis)
19. [Limits and Quotas](#19-limits-and-quotas)
20. [Pricing Model](#20-pricing-model)
21. [Best Practices](#21-best-practices)
22. [Common Failure Modes and How to Think About Them](#22-common-failure-modes-and-how-to-think-about-them)

---

## 1. What is Amazon API Gateway?

Amazon API Gateway is a **fully managed service** from AWS that acts as the entry point — the front door — for any application that wants to expose functionality over HTTP or WebSocket. It is not an application server; it is a **managed reverse proxy with programmable behavior** layered on top.

Its job is to:

- Accept incoming requests from any client (browser, mobile app, IoT device, another service)
- Optionally authenticate and authorize that request
- Optionally validate, transform, or enrich the request payload
- Route the request to a backend (Lambda, EC2, an external URL, or even an AWS service directly)
- Receive the backend response, optionally transform it
- Return the response to the client

All of this happens without you managing any server infrastructure. API Gateway scales automatically, handles TLS termination, and integrates natively with the AWS ecosystem.

### What It Is Not

API Gateway is not a load balancer (though it distributes traffic). It is not an application framework. It does not run your business logic — it mediates the conversation between clients and wherever your logic lives.

---

## 2. Where It Fits in Modern Architecture

### Serverless API Pattern

The most common pattern today is: Client → API Gateway → Lambda → Database. This stack is fully serverless, infinitely scalable within AWS limits, and requires zero server management.

```
Client
  └─▶ API Gateway          (routing, auth, rate limiting)
        └─▶ Lambda         (business logic)
              └─▶ DynamoDB / RDS / S3  (data)
```

### Microservices Gateway Pattern

API Gateway can serve as a unified entry point for many backend services, routing different paths to different backends:

```
Client
  └─▶ API Gateway
        ├─▶ /users      → Lambda (user service)
        ├─▶ /orders     → ECS container (order service)
        ├─▶ /payments   → EC2 / external payment API
        └─▶ /inventory  → Another Lambda (inventory service)
```

### Backend for Frontend (BFF) Pattern

Different API Gateway instances (or stages) serve different clients — a mobile BFF, a web BFF — with tailored response shapes and auth mechanisms.

### Direct AWS Service Integration Pattern

API Gateway can call AWS services directly (DynamoDB, SQS, SNS, S3, Step Functions) without any Lambda in between, reducing cost and latency for simple operations:

```
Client → API Gateway → DynamoDB (PutItem directly)
```

---

## 3. API Types — REST vs HTTP vs WebSocket

Amazon API Gateway offers three fundamentally different API flavors.

### REST API (v1)

The original and most feature-rich API type. REST APIs support:

- Full request/response transformation via Velocity Template Language (VTL)
- Response caching at the API Gateway layer
- Usage plans and API keys for monetization and throttling per consumer
- Request validation via JSON Schema models
- Edge-optimized, regional, and private endpoint types
- Direct AWS service integrations
- WAF (Web Application Firewall) integration
- Custom gateway responses (customize error messages)

REST APIs have higher latency and cost compared to HTTP APIs due to the additional processing overhead.

### HTTP API (v2)

Introduced to address the cost and latency overhead of REST APIs. HTTP APIs are purpose-built for:

- Lambda proxy integrations
- HTTP backend proxy integrations
- JWT-based authorization (OIDC/OAuth 2.0) natively
- Lower cost (roughly 70% cheaper than REST APIs)
- Lower latency

What HTTP APIs sacrifice:

- No VTL mapping templates (no request/response transformation)
- No per-method caching
- No usage plans or API keys
- No edge-optimized endpoints
- No direct AWS service integrations
- No X-Ray tracing (at time of writing)

**Decision rule:** If you need transformation, caching, or usage plans → REST API. If you need a simple, fast, cheap Lambda or HTTP proxy → HTTP API.

### WebSocket API

A completely different communication model. Instead of stateless request/response cycles, WebSocket APIs maintain a persistent, bidirectional connection between client and server.

Key properties:

- The connection persists until either side closes it
- Both client and server can send messages at any time without a request/response cycle
- API Gateway assigns each connection a unique Connection ID
- Routes are selected based on a field in the message body (the route selection expression)
- Three built-in routes: `$connect`, `$disconnect`, `$default`
- You define custom routes for different message types (e.g., `sendMessage`, `typing`, `subscribe`)

WebSocket APIs are ideal for: chat applications, live dashboards, collaborative tools, multiplayer games, real-time notifications.

---

## 4. Core Concepts Deep Dive

### API

The top-level container. It has a unique ID (e.g., `abc123xyz`) and an auto-generated invoke URL in the pattern: `https://{apiId}.execute-api.{region}.amazonaws.com/{stage}`

### Resource

A resource is a URL path segment. Resources form a tree. The root is `/`. You build paths by nesting resources:

```
/                      (root resource)
├── /users             (resource)
│   └── /{userId}      (child resource with path parameter)
│       └── /orders    (grandchild resource)
└── /products
```

Path parameters like `{userId}` are dynamic — they capture whatever value appears in that position of the URL and make it available to the integration.

### Method

A method is an HTTP verb (GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS, ANY) attached to a resource. Each method has its own independent configuration: authorization type, integration, request validators, and so on.

The method is the fundamental unit of API behavior.

### Integration

The integration is the connection between a method and a backend. It defines where the request goes and how. See Section 6 for the full breakdown.

### Model

A model is a JSON Schema document that describes the shape of a request body or response body. Models enable:

- Request validation (reject malformed requests before they hit your backend)
- Documentation generation
- SDK generation

### Authorizer

An authorizer is a pluggable authentication/authorization component. When a request arrives, the authorizer runs first and either allows or denies the request from reaching the backend. See Section 9.

### Stage

A stage is a named snapshot of your API's deployed configuration. Think of it as an environment: `dev`, `staging`, `prod`. Each stage has its own URL, its own settings (logging level, caching, throttling), and its own stage variables. See Section 10.

### Deployment

A deployment is an immutable snapshot of your API resources, methods, and integrations at a point in time. Deploying pushes a snapshot to a stage. The API itself is a "draft" until deployed.

### Stage Variables

Stage variables are key-value pairs scoped to a stage. They behave like environment variables and can be referenced in integration URIs, mapping templates, and Lambda ARNs. This allows a single API definition to behave differently per stage (e.g., point to different Lambda aliases, different backend URLs).

---

## 5. Endpoint Types

### Regional

The API is deployed in a specific AWS region and accessed directly. DNS resolves to that region. Best for:

- APIs consumed by clients in the same or nearby region
- APIs you want to put behind your own CloudFront distribution for custom caching behavior

### Edge-Optimized

The API endpoint is fronted by the AWS CloudFront global network. Incoming requests are routed to the nearest CloudFront point of presence (PoP) and then traverse the AWS private backbone to the region where the API lives. This reduces latency for globally distributed clients.

Important nuance: the API Lambda/backend still runs in a single region. Only the HTTP connection from client to CloudFront is optimized. If the backend itself has regional latency, edge-optimization won't help beyond the first hop.

### Private

The API is accessible only from within a VPC, via a VPC Endpoint (Interface Endpoint for `execute-api`). No public internet exposure. Suitable for internal microservices, backend APIs that should never be publicly reachable.

Private APIs use resource policies to further control which VPCs or VPC endpoints can access them.

---

## 6. Integration Types

The integration is the bridge between API Gateway and your backend. There are five integration types.

### AWS_PROXY (Lambda Proxy)

API Gateway passes the entire HTTP request to Lambda in a structured event object and expects Lambda to return a structured response object. API Gateway does zero transformation — it is a transparent proxy.

The event Lambda receives contains:

- `httpMethod` — the HTTP verb
- `path` — the full resource path
- `pathParameters` — captured path parameter values
- `queryStringParameters` — parsed query string
- `headers` — all HTTP headers
- `body` — the raw request body as a string (not parsed)
- `requestContext` — metadata about the request, stage, identity, authorizer context
- `isBase64Encoded` — whether body is base64 encoded (for binary payloads)

Lambda must return:

```json
{
  "statusCode": 200,
  "headers": { "Content-Type": "application/json" },
  "body": "{\"message\": \"hello\"}"
}
```

The `body` field must always be a string. JSON bodies must be serialized with `JSON.stringify()`.

This is the most common integration type for Lambda-backed APIs.

### AWS (Lambda Non-Proxy / AWS Service)

API Gateway can call any AWS service action (Lambda, DynamoDB, SQS, SNS, S3, Step Functions, and more) using this integration type. Unlike AWS_PROXY, you have full control over what gets sent to the backend and how the response is returned, using VTL mapping templates.

For Lambda non-proxy, you define exactly what JSON body gets passed to Lambda. For DynamoDB, you construct the DynamoDB API call (e.g., `PutItem`, `GetItem`) directly in the mapping template.

This integration type gives maximum flexibility but requires writing VTL.

### HTTP_PROXY

Forwards the incoming request as-is to an external HTTP/HTTPS endpoint. API Gateway acts as a transparent pass-through to any HTTP backend — a server on EC2, an ECS service, an external third-party API. No transformation.

### HTTP (Non-Proxy)

Like HTTP_PROXY but with VTL-based transformation of the request before sending and the response after receiving. Useful when you need to adapt the shape of the request/response between client and backend.

### MOCK

Returns a hardcoded response without calling any backend. Useful for:

- Building the API contract before backends are ready
- Testing client integrations
- Returning static responses for certain routes (e.g., a health check endpoint)

---

## 7. The Request Lifecycle

Understanding the full lifecycle of a request through API Gateway is essential. Every request passes through four conceptual phases.

### Phase 1 — Method Request

This is the API Gateway's front door. It handles:

- **Authentication/Authorization**: The authorizer runs here. If it denies the request, processing stops and a 403 is returned immediately.
- **Request Validation**: If a request validator is configured, API Gateway checks query string parameters, headers, and the request body against the configured model. A failed validation returns 400 before the backend is ever called.
- **API Key Check**: If the method requires an API key, API Gateway validates it against usage plans here.

### Phase 2 — Integration Request

This phase prepares the request to be sent to the backend:

- For proxy integrations, the request is passed through with minimal modification
- For non-proxy integrations, a VTL mapping template transforms the request body, headers, and path into the format the backend expects
- Path parameters, query strings, and headers can be mapped to backend request fields

### Phase 3 — Integration Response

After the backend responds, this phase processes that response:

- For proxy integrations, the response from the backend is passed through unchanged
- For non-proxy integrations, a VTL mapping template transforms the backend response into the desired client-facing format
- Different HTTP status codes from the backend can be mapped to different status codes for the client
- Response headers can be added, modified, or removed

### Phase 4 — Method Response

The final shape of what the client receives:

- HTTP status codes the API is allowed to return are defined here (200, 400, 404, 500, etc.)
- Response headers and their types are declared
- Response body models are associated with status codes

### Error Handling in the Lifecycle

If any phase fails (authorization denied, validation failed, backend unreachable, timeout), API Gateway returns a **gateway response** — a predefined error response. These can be customized per error type (e.g., change the body of a 403, add headers to a 429).

---

## 8. Request and Response Transformation (VTL)

### What is VTL?

Velocity Template Language is a templating language originally from the Apache Velocity project. API Gateway uses it in integration request and response mapping templates to transform payloads.

VTL is only available in REST APIs. HTTP APIs have no transformation capability.

### The Input Object

Inside a VTL template, `$input` provides access to the incoming data:

- `$input.body` — the raw request body as a string
- `$input.json('$.field')` — extracts a JSON field from the body
- `$input.path('$.field')` — parses the body as JSON and navigates it
- `$input.params('name')` — retrieves a path parameter or query string value

### The Context Object

`$context` gives access to request metadata:

- `$context.requestId` — unique request identifier
- `$context.identity.sourceIp` — client IP
- `$context.authorizer.principalId` — user ID returned by authorizer
- `$context.stage` — current stage name
- `$context.httpMethod` — HTTP method

### The Util Object

`$util` provides helper functions:

- `$util.escapeJavaScript(str)` — escapes a string for JavaScript
- `$util.urlEncode(str)` / `$util.urlDecode(str)`
- `$util.base64Encode(str)` / `$util.base64Decode(str)`
- `$util.parseJson(str)` — parses a JSON string into an object

### Example — Extracting Fields for a DynamoDB PutItem

```velocity
{
  "TableName": "Users",
  "Item": {
    "userId": { "S": "$context.requestId" },
    "name":   { "S": "$input.json('$.name')" },
    "email":  { "S": "$input.json('$.email')" },
    "createdAt": { "S": "$context.requestTime" }
  }
}
```

### Example — Reshaping a DynamoDB Response

```velocity
#set($item = $input.path('$.Item'))
{
  "userId": "$item.userId.S",
  "name": "$item.name.S",
  "email": "$item.email.S"
}
```

### Passthrough Behavior

When no mapping template matches the incoming Content-Type, API Gateway can either:

- **WHEN_NO_MATCH** — pass the body through as-is (default for most integrations)
- **WHEN_NO_TEMPLATES** — pass through only when no templates are defined at all
- **NEVER** — reject the request if no template matches (strict mode)

---

## 9. Authorization and Security Model

### The Layered Security Model

Security in API Gateway is layered. Multiple mechanisms can coexist and apply at different levels.

```
Request arrives
    │
    ▼
Resource Policy          (IP allowlist/denylist, VPC access, cross-account)
    │
    ▼
API Key Validation       (if method requires API key)
    │
    ▼
Method-Level Authorizer  (IAM, Cognito, Lambda)
    │
    ▼
Backend (Lambda, etc.)   (can do additional authorization checks)
```

### IAM Authorization (AWS_IAM)

The client must sign every request using AWS Signature Version 4 (SigV4). This is the same signing mechanism used for all AWS API calls.

The IAM principal (user or role) making the request must have an IAM policy granting `execute-api:Invoke` on the specific API resource ARN.

This is ideal for service-to-service calls within AWS (Lambda calling Lambda via API Gateway, an EC2 instance calling an internal API). It leverages existing IAM roles and requires no additional token management.

The ARN format for execute-api permissions:

```
arn:aws:execute-api:{region}:{accountId}:{apiId}/{stage}/{httpMethod}/{resource}
```

Wildcards are supported:

```
arn:aws:execute-api:us-east-1:123456789012:abc123/*/*/*
```

(grants access to all methods and all resources in all stages of this API)

### Cognito User Pool Authorizer

The client authenticates against an Amazon Cognito User Pool and receives a JWT (ID token or access token). The client includes this token in the `Authorization` header. API Gateway validates the token against the configured User Pool without any Lambda involvement — validation is built in.

What API Gateway validates:

- The token signature (using the pool's public keys)
- The token expiration (`exp` claim)
- The token audience (`aud` claim — must match the configured App Client ID)
- The token issuer (`iss` claim — must match the User Pool URL)

What API Gateway does NOT validate:

- Custom claims or scopes (you validate those in your Lambda)
- Whether the user is allowed to access specific resources (that's your business logic)

### Lambda Authorizer (Custom Authorizer)

A Lambda function you write that receives the request (or a token from the request) and returns an IAM policy document that either allows or denies the request.

Two sub-types:

**Token-based**: The authorizer receives a single token (typically from the `Authorization` header). It validates that token (against a JWT library, a database, a third-party auth service, anything) and returns a policy. The token value is the only input.

**Request-based**: The authorizer receives the full request context — headers, query strings, stage variables, path parameters. This allows authorization decisions based on multiple inputs simultaneously (e.g., the API key AND the user role AND the requested resource).

The returned IAM policy controls access via `execute-api:Invoke`. The `Resource` field in the policy can be specific (one method) or use wildcards (all methods in the API). The policy can also include a `context` object — key-value pairs that are passed downstream to the backend via `$context.authorizer.*`.

**Authorizer caching**: By default, the authorizer result is cached for up to 3600 seconds (configurable down to 0). The cache key is the token value (for token-based) or a combination of identity sources (for request-based). Caching reduces the number of Lambda invocations for authorization.

### Resource Policies

Resource policies are JSON IAM policies attached directly to the API. They control access before any authorizer runs. Common uses:

- Restrict API access to specific IP address ranges
- Allow access only from specific VPCs or VPC Endpoints
- Grant cross-account access (allow a specific AWS account to call your API)
- Deny specific IPs while allowing everything else

Resource policies and authorizers work together. For private APIs, both the resource policy AND the authorizer must allow the request.

### AWS WAF Integration

API Gateway integrates with AWS Web Application Firewall. WAF sits in front of your API (for REST APIs and regional HTTP APIs) and can:

- Block based on IP reputation lists
- Protect against SQL injection and XSS
- Rate-limit by IP
- Block based on geographic location
- Apply custom rules based on request attributes

WAF rules are evaluated before requests reach API Gateway, providing an additional security layer beyond what API Gateway itself offers.

---

## 10. Stages, Deployments, and Versioning

### The Deployment Model

This is one of the most important things to understand about API Gateway: **changes to an API are not live until deployed**.

The API definition (resources, methods, integrations, authorizers) is a draft. When you deploy, API Gateway takes a snapshot of the current API definition and associates it with a stage. The stage URL then serves traffic based on that snapshot.

This separation between API definition and deployed state enables:

- Making multiple changes to the API before releasing them
- Deploying to `dev` first, testing, then deploying to `prod`
- Rolling back by pointing a stage to a previous deployment

### Stages as Environments

A stage is more than just a deployment target. Each stage has independent configuration:

- **Throttling settings** — different rate limits per stage
- **Caching settings** — enable/disable and configure cache per stage
- **Logging settings** — different log levels per stage (INFO for dev, ERROR for prod)
- **Stage variables** — per-stage key-value pairs
- **X-Ray tracing** — enable/disable per stage
- **WAF association** — different WAF rules per stage

### Stage Variables

Stage variables deserve special attention. They allow a single API definition to behave differently per stage without changing the API itself.

Common patterns:

**Lambda alias routing**: Instead of hardcoding a Lambda ARN in the integration, use:

```
arn:aws:lambda:{region}:{accountId}:function:MyFunction:${stageVariables.lambdaAlias}
```

Set `lambdaAlias` to `DEV` in the dev stage and `PROD` in the prod stage. Lambda aliases then point to different function versions.

**Backend URL**: Store the backend base URL in a stage variable and reference it in HTTP integrations. Changing the backend for a stage requires only updating the stage variable, not redeploying the API.

**Feature flags**: Reference stage variables in mapping templates to conditionally modify behavior per stage.

### Canary Releases

API Gateway supports canary deployments natively at the stage level. When configured, a percentage of traffic is routed to a new "canary" deployment while the rest continues to the base deployment.

Canary settings:

- **Percent traffic**: what fraction (0–100%) goes to the canary
- **Canary deployment**: which deployment snapshot the canary uses
- **Override stage variables**: the canary can use different stage variables than the base

After validating the canary, you promote it (making it the base) or roll it back (removing the canary). This is a built-in blue/green deployment mechanism.

### API Versioning Strategies

API versioning is not built into API Gateway as a first-class feature. You implement it yourself using common patterns:

**URL path versioning**: Different base paths on the same API or different APIs:

```
/v1/users
/v2/users
```

Simple, explicit, visible. The most common approach.

**Stage-based versioning**: Use stages as version identifiers (`v1` stage, `v2` stage). Works but conflates environments (dev/prod) with API versions.

**Custom domain + base path mapping**: Map `api.example.com/v1` to one API/stage and `api.example.com/v2` to another. Cleanest approach — version is in the URL but the underlying API structure is separate.

**Header-based versioning**: Clients send `API-Version: 2` in a header. API Gateway routes based on this using Lambda authorizer context or VTL. More flexible but harder to cache and debug.

---

## 11. Usage Plans, API Keys, and Throttling

### The Throttling Model

API Gateway has throttling at multiple levels, applied in order. Understanding this hierarchy is critical.

**Account-level**: A hard ceiling per AWS account per region. Default is 10,000 RPS (requests per second) with a burst of 5,000. This is a shared limit across ALL APIs in the account/region. Exceeding it returns 429 for all APIs.

**Stage-level**: Configure a default rate limit and burst limit for all methods in a stage. Applies after the account limit is checked.

**Method-level**: Override throttling for a specific route (e.g., `/search` might have a lower limit than `/users` because it's more expensive).

**Usage plan-level**: Apply per API-key throttling and quotas. This is the consumer-level control — different API keys can have different limits.

The effective limit for any request is the minimum of all applicable limits.

### How Throttling Works (Token Bucket Algorithm)

API Gateway uses a token bucket algorithm:

- The bucket has a maximum capacity equal to the burst limit
- The bucket fills at the steady-state rate (RPS) — this many tokens are added per second
- Each request consumes one token
- If the bucket is empty, the request is rejected with 429

This means short bursts above the steady-state rate are allowed (up to the burst limit), but sustained traffic must stay at or below the steady-state rate.

### Usage Plans

A usage plan defines throttling and quota settings that are associated with one or more API stages and applied to specific API keys.

Components of a usage plan:

- **Throttle**: rate limit (RPS) and burst limit for this plan
- **Quota**: maximum number of requests per day/week/month
- **Associated stages**: which API stages this plan applies to

A consumer (API key) is associated with a usage plan. When they call the API, their requests are counted against the plan's quota and throttled according to the plan's limits.

This enables monetization scenarios (free tier vs. paid tier, different service tiers for different customers) and abuse prevention.

### API Keys

API keys are alphanumeric strings (generated by API Gateway or imported) that identify a consumer. They are passed in the `x-api-key` request header.

Important: API keys are NOT a security mechanism for authentication. They identify a consumer for the purposes of usage tracking and throttling. They should not be used as a substitute for authentication (use Cognito or Lambda authorizers for that). API keys can be used alongside authorizers.

---

## 12. Caching

API Gateway can cache integration responses at the stage level, reducing the number of calls to your backend and improving latency for repeated identical requests.

### How It Works

When caching is enabled, API Gateway checks whether the incoming request matches a cached response. If it does (a cache hit), it returns the cached response immediately without calling the backend. If not (a cache miss), it calls the backend and stores the response in the cache for future requests.

### Cache Key

By default, the cache key is the full request path (including path parameters). You can extend the cache key to include:

- Specific query string parameters (only the ones you designate, not all of them)
- Specific request headers

This is important: if you have `/users?role=admin&format=json`, and you add `role` to the cache key but not `format`, then the cache treats `?role=admin&format=xml` as the same cache entry as `?role=admin&format=json`.

### Cache TTL

Configurable from 0 to 3600 seconds (1 hour). Default is 300 seconds. Can be set at the stage level (applies to all methods) and overridden per method.

### Cache Invalidation

Clients can bypass the cache by sending `Cache-Control: max-age=0` in the request header. You can require that this capability be protected by IAM permission (`execute-api:InvalidateCache`) to prevent unauthorized cache busting.

Programmatic invalidation of the entire cache (or per-key) is also possible via the API.

### Cache Encryption

Cached data can be encrypted at rest. This is important for APIs that cache sensitive data.

### Cost and Sizing

Caching has an additional hourly cost based on the cache size. Cache sizes range from 0.5 GB to 237 GB. Larger caches cost more but have higher hit rates for larger data sets.

Caching is only available for REST APIs. HTTP APIs do not support caching.

---

## 13. CORS

Cross-Origin Resource Sharing (CORS) is a browser security mechanism that restricts web pages from making requests to a different domain than the one that served the page. Understanding CORS is critical for any API consumed by browser-based applications.

### The CORS Flow

When a browser wants to make a cross-origin request, it first sends a **preflight request** — an HTTP `OPTIONS` request to the API endpoint. The server must respond with headers indicating what origins, methods, and headers are allowed. If the response is satisfactory, the browser proceeds with the actual request.

The relevant response headers:

- `Access-Control-Allow-Origin` — which origins are permitted (`*` for all, or a specific domain)
- `Access-Control-Allow-Methods` — which HTTP methods are allowed
- `Access-Control-Allow-Headers` — which request headers are allowed
- `Access-Control-Max-Age` — how long the browser should cache the preflight result
- `Access-Control-Allow-Credentials` — whether cookies/credentials are included

### What API Gateway Does

API Gateway must be configured to respond to `OPTIONS` requests on each resource. It must also ensure the actual response from your backend (Lambda, etc.) includes the `Access-Control-Allow-Origin` header.

This is a common point of confusion: the preflight is handled by API Gateway, but the actual request's CORS headers come from your backend (Lambda). Both must be configured correctly.

If you use Lambda Proxy integration, your Lambda must return the CORS headers in every response. If you use non-proxy integration, you add the headers in the integration response mapping template.

### `Access-Control-Allow-Origin: *`

Using a wildcard is convenient but comes with restrictions: you cannot send credentials (cookies, Authorization headers) with wildcard origins. For authenticated APIs, you must specify the exact allowed origin(s).

---

## 14. Logging, Monitoring, and Observability

### Execution Logs

API Gateway can write detailed execution logs to CloudWatch Logs. These logs capture the full lifecycle of each request: the incoming request (headers, body), the integration request sent to the backend, the integration response, and the final method response.

Log levels:

- **ERROR** — only log requests that fail
- **INFO** — log every request in detail

Execution logs are written to a log group named `API-Gateway-Execution-Logs_{apiId}/{stageName}`.

Execution logs are verbose and expensive for high-traffic APIs. Use ERROR level in production and INFO level when debugging.

### Access Logs

Access logs are a customizable, one-line-per-request log in whatever format you choose (JSON, CLF, CSV). They are written to a log group you specify. This is the log format suitable for analysis tools, log aggregators (Splunk, Datadog), and business analytics.

You compose the log format using context variables from `$context.*`. A typical JSON access log format:

```json
{
  "requestId": "$context.requestId",
  "ip": "$context.identity.sourceIp",
  "caller": "$context.identity.caller",
  "user": "$context.identity.user",
  "requestTime": "$context.requestTime",
  "httpMethod": "$context.httpMethod",
  "resourcePath": "$context.resourcePath",
  "status": "$context.status",
  "protocol": "$context.protocol",
  "responseLength": "$context.responseLength",
  "integrationLatency": "$context.integrationLatency",
  "responseLatency": "$context.responseLatency",
  "errorMessage": "$context.error.message"
}
```

### CloudWatch Metrics

API Gateway publishes metrics to CloudWatch automatically. Key metrics:

**Count**: Total number of API calls. Use to track traffic volume and trends.

**Latency**: The total time from when API Gateway receives a request to when it sends the response back to the client. This includes integration latency plus API Gateway processing overhead.

**IntegrationLatency**: The time from when API Gateway sends the request to the backend to when it receives the response. This is your backend performance metric.

**4XXError**: Client errors (bad requests, unauthorized, not found, rate limited). A spike in 4XX often indicates a client bug, an auth issue, or someone probing your API.

**5XXError**: Server errors (Lambda crashes, timeouts, integration failures). Any spike in 5XX needs immediate investigation.

**CacheHitCount / CacheMissCount**: Caching effectiveness metrics. A low hit rate suggests the cache TTL is too short, the cache key includes too many varying parameters, or the traffic pattern is highly random.

Metrics can be filtered by API, stage, or resource/method.

### AWS X-Ray

When X-Ray tracing is enabled, API Gateway emits trace segments for every request. Combined with X-Ray instrumentation in your Lambda functions and any downstream services, you get end-to-end distributed traces showing exactly where time is being spent across your entire request path.

X-Ray is invaluable for latency analysis and understanding the distribution of time between API Gateway overhead, Lambda cold start, Lambda execution, and downstream service calls.

---

## 15. Custom Domain Names and TLS

### Why Custom Domains

The default API Gateway URL is functional but not user-friendly or brand-appropriate: `https://abc123xyz.execute-api.us-east-1.amazonaws.com/prod/users`

Custom domains allow you to expose your API at: `https://api.mycompany.com/v1/users`

### How It Works

1. You obtain a TLS certificate for your domain from AWS Certificate Manager (ACM). For edge-optimized APIs, the certificate must be in `us-east-1`. For regional APIs, it must be in the same region as the API.
    
2. You create a custom domain name resource in API Gateway, associating it with the ACM certificate.
    
3. API Gateway gives you a CloudFront distribution domain (for edge-optimized) or a regional API Gateway domain. You create a DNS CNAME or Alias record pointing your custom domain to this target.
    
4. You create a base path mapping — associating the custom domain with a specific API and stage, and optionally at a specific base path (e.g., `/v1`).
    

### Base Path Mappings

Base path mappings allow a single custom domain to serve multiple APIs:

```
api.mycompany.com/v1  → REST API (id: abc123, stage: prod)
api.mycompany.com/v2  → REST API (id: xyz789, stage: prod)
api.mycompany.com/ws  → WebSocket API (id: ws456, stage: prod)
```

The base path is stripped before the request reaches the API, so your API resources don't need to include the base path prefix.

### TLS Versions

API Gateway supports TLS 1.0, 1.2, and 1.3. You configure the minimum TLS version in the custom domain settings. For modern security standards, requiring TLS 1.2 or higher is recommended. TLS 1.0 should be disabled.

---

## 16. WebSocket APIs In Depth

### Connection Lifecycle

A WebSocket connection goes through three phases:

**Connect**: Client initiates a WebSocket handshake. API Gateway generates a unique Connection ID. The `$connect` route Lambda runs. Here you authenticate the client (via query string token, since WebSocket doesn't support custom headers in the browser) and store the Connection ID for later use.

**Communication**: Client and server exchange messages. Each message triggers a route Lambda based on the route selection expression. Messages can flow in both directions: client sends a message (triggers a Lambda), and the Lambda can push messages back to the client or to any other connected client (using the Management API).

**Disconnect**: Either side closes the connection. The `$disconnect` route Lambda runs. Clean up the Connection ID from storage.

### Route Selection Expression

The route selection expression is evaluated against each incoming message to determine which route handles it. The most common pattern is `$request.body.action`, meaning the `action` field in the JSON message body determines routing.

If a client sends `{"action": "sendMessage", "text": "Hello"}`, API Gateway routes to the `sendMessage` route. If no matching route exists, the `$default` route handles it.

### The Management API

This is what enables server-to-client push. From any Lambda (including one triggered by an API call to a separate REST API), you can push a message to any connected WebSocket client if you know their Connection ID.

The endpoint format is:

```
https://{apiId}.execute-api.{region}.amazonaws.com/{stage}/@connections/{connectionId}
```

Operations via the Management API:

- **POST** to send a message to the connection
- **GET** to check if the connection is still active (and get connection metadata)
- **DELETE** to forcibly disconnect a client

This is the mechanism for broadcast (send to all connections), targeted messaging (send to specific users), and pub/sub patterns (maintain topic subscriptions in DynamoDB).

### Connection Storage Pattern

API Gateway does not maintain state about connections beyond routing messages to Lambdas. You must store Connection IDs yourself (typically in DynamoDB) to:

- Know which clients are connected
- Map users to connections (user ID → Connection ID)
- Implement group/room membership for targeted messaging
- Recover from Lambda cold starts (the Lambda needs to look up who to notify)

### Idle Connection Timeout

API Gateway disconnects idle WebSocket connections after 10 minutes by default. The maximum is 2 hours. To keep connections alive, clients should send periodic ping messages, and your server should handle them (typically via the `$default` route).

---

## 17. HTTP APIs (v2) In Depth

### Architectural Differences from REST APIs

HTTP APIs have a fundamentally simpler architecture:

- No method-level configuration — routes are defined directly (e.g., `GET /users`)
- No separate deployment step for most changes — stages can auto-deploy
- No integration request/response configuration — pure proxy behavior
- No VTL — payload passes through unchanged
- Simplified authorizer model — JWT natively, or Lambda authorizer with a simpler contract

### Lambda Payload Format Versions

HTTP APIs with Lambda integration support two payload format versions. This is a critical distinction.

**Format 1.0** (same as REST API Lambda Proxy): Maintains backward compatibility with existing Lambda functions written for REST API.

**Format 2.0** (new, recommended): A cleaner, more efficient event structure. Key differences:

- `cookies` is a separate array instead of being in headers
- `headers` are lowercased
- `queryStringParameters` handles multi-value params differently
- `requestContext` has a cleaner structure
- The response can be simplified — just return a string body and API Gateway wraps it in a 200 response automatically

Format 2.0 is the default for new HTTP APIs and is recommended for new Lambda functions.

### JWT Authorizers

HTTP APIs natively support JWT validation without any Lambda. You configure:

- **Issuer**: The token issuer URL (e.g., a Cognito User Pool URL, Auth0 domain, Okta domain)
- **Audiences**: Valid audience values (list of allowed `aud` claim values)
- **Identity source**: Where in the request the token is found (default: `$request.header.Authorization`)

API Gateway fetches the issuer's JWKS (JSON Web Key Set) endpoint to get the public keys, validates the signature, checks expiration, and validates the audience. All built-in. No Lambda required.

For authorization beyond authentication (checking scopes, roles, custom claims), you still need Lambda.

### Route and Integration Simplicity

HTTP API routes are defined with a direct verb+path syntax: `GET /users`, `POST /users`, `GET /users/{userId}`. You can also define `$default` as a catch-all route.

Integrations are simpler too — for Lambda, you specify the Lambda ARN and the payload format version. That's it. No integration request/response configuration needed.

### Auto-Deploy

HTTP API stages support auto-deploy, which means every time you change a route or integration, the changes are immediately live on that stage. This is convenient for development but means you must be careful about deploying breaking changes directly to production stages.

---

## 18. VPC Integration and Private APIs

### VPC Link

VPC Link allows API Gateway to connect to resources inside a private VPC — resources that have no public internet endpoint. This is how you use API Gateway as a front door for services running on EC2, ECS, EKS, or an Application Load Balancer inside a VPC.

For REST APIs, VPC Link uses a Network Load Balancer (NLB) inside the VPC as the target. API Gateway integrates with the NLB, and the NLB routes to your private resources.

For HTTP APIs, VPC Link is simpler and directly targets Application Load Balancers, Network Load Balancers, AWS Cloud Map services, or IP addresses within the VPC.

### Private APIs

Private APIs are only accessible from within a VPC via an Interface VPC Endpoint for `execute-api`. No traffic ever leaves the AWS network.

The architecture:

1. Create an Interface VPC Endpoint for `execute-api` in your VPC
2. Create a private API Gateway with `PRIVATE` endpoint type
3. Attach a resource policy that grants access only from the specific VPC or VPC endpoint

Private APIs are ideal for internal microservices that should never be exposed to the public internet, internal admin APIs, and APIs called by other services running inside the same VPC or connected VPCs.

### DNS Resolution for Private APIs

Private APIs have a tricky DNS behavior. The default API Gateway URL (`{apiId}.execute-api.{region}.amazonaws.com`) does not resolve within a VPC unless VPC Endpoint private DNS is enabled or you use the VPC Endpoint DNS name directly with a `Host` header.

Custom domain names simplify this — you can create a Route 53 private hosted zone with a CNAME pointing your custom domain to the VPC Endpoint, making the private API accessible at a clean URL within the VPC.

---

## 19. Limits and Quotas

Understanding limits is essential for capacity planning. These are the key limits as of mid-2024.

### Account-Level Limits (Adjustable via Support)

| Limit                                | Default          |
| ------------------------------------ | ---------------- |
| Regional APIs per account per region | 600              |
| Edge-optimized APIs per account      | 120              |
| Stages per API                       | 10               |
| Resources per REST API               | 300              |
| Methods per resource                 | All HTTP methods |
| Authorizers per API                  | 10               |
| Usage plans per account              | 300              |
| API keys per account                 | 500              |

### Request Limits (Adjustable via Support)

|Limit|Default|
|---|---|
|Steady-state requests per second (account)|10,000 RPS|
|Burst (account)|5,000 requests|
|Maximum integration timeout (REST API)|29 seconds|
|Maximum integration timeout (HTTP API)|30 seconds|
|Maximum integration timeout (WebSocket)|29 seconds|

### Fixed Limits (Not Adjustable)

|Limit|Value|
|---|---|
|Maximum payload size (REST API)|10 MB|
|Maximum payload size (HTTP API)|10 MB|
|Maximum WebSocket message size|128 KB (per frame) / 32 KB (per message)|
|Maximum authorizer cache TTL|3600 seconds|
|Cache TTL maximum|3600 seconds|
|Maximum base path mappings per custom domain|200|
|Maximum VPC Links per account|20|

### Integration Timeout

The integration timeout is critical: if your backend takes longer than the timeout to respond, API Gateway returns a 504 Gateway Timeout. The maximum is 29 seconds for REST APIs — well below Lambda's maximum execution time of 15 minutes.

This means API Gateway is not suitable for long-running synchronous operations. For long-running work, the pattern is: API Gateway → Lambda → SQS/Step Functions (async), with a separate mechanism for the client to poll for the result.

---

## 20. Pricing Model

API Gateway pricing has three dimensions: call volume, data transfer, and optional features (cache).

### REST API Pricing

Charged per million API calls received. The price decreases at volume tiers (tiered pricing within a month):

- First 333 million calls/month: $3.50 per million
- Next 667 million: $2.80 per million
- Next 19 billion: $2.38 per million
- Over 20 billion: $1.51 per million

Additional charges:

- **Cache**: hourly charge based on cache size ($0.02/hr for 0.5 GB to $3.80/hr for 237 GB)
- **Private API**: additional $0.01 per GB for data processed

### HTTP API Pricing

- First 300 million calls/month: $1.00 per million
- Over 300 million: $0.90 per million

No caching charges (no caching feature). Significantly cheaper than REST APIs.

### WebSocket API Pricing

- **Connection minutes**: $0.29 per million connection minutes
- **Messages**: $1.00 per million messages (up to 32 KB each; larger messages are charged as multiple messages)

### Data Transfer

Standard AWS data transfer rates apply to data transferred out of API Gateway.

### Free Tier

The AWS Free Tier includes:

- 1 million REST API calls per month for 12 months
- 1 million HTTP API calls per month for 12 months
- 1 million WebSocket messages and 750,000 connection minutes per month for 12 months

All prices are for US East (N. Virginia) and vary slightly by region.

---

## 21. Best Practices

### API Design

Use nouns for resources, not verbs. The HTTP method is the verb:

```
GET    /users          — list users
POST   /users          — create a user
GET    /users/{id}     — get a specific user
PUT    /users/{id}     — replace a user
PATCH  /users/{id}     — partially update a user
DELETE /users/{id}     — delete a user
```

Version your API from day one. Breaking changes to an existing version will break clients. Use path-based versioning (`/v1/`, `/v2/`) for simplicity and visibility.

Return meaningful HTTP status codes. Not everything is `200 OK`. `201 Created` for successful POST, `204 No Content` for successful DELETE, `400 Bad Request` for validation errors, `401 Unauthorized` for unauthenticated, `403 Forbidden` for authenticated but not authorized, `404 Not Found` for missing resources, `409 Conflict` for state conflicts, `429 Too Many Requests` for throttling, `503 Service Unavailable` for downstream outages.

Return consistent error response bodies. Every error response should have the same shape so clients can handle them uniformly:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The 'email' field is required",
    "requestId": "abc-123-xyz"
  }
}
```

### Security

Never disable authorization on sensitive endpoints. Every endpoint that accesses data or performs operations should have an authorizer.

Validate all inputs at the API Gateway level using Models. Reject malformed requests before they reach your Lambda — it saves compute cost and prevents injection attacks from reaching your business logic.

Use the principle of least privilege for all IAM roles. The Lambda execution role should only have permissions to the specific resources it needs. The API Gateway execution role should only have permissions to invoke the specific Lambda functions.

Enable WAF for any public-facing API. At minimum, use AWS Managed Rules to protect against OWASP Top 10 vulnerabilities.

Use resource policies to restrict access when possible. If your API is consumed by a known set of IPs or VPCs, lock it down.

Rotate API keys regularly if you use them. Treat them like passwords.

### Performance

Choose HTTP APIs over REST APIs when you don't need the additional features. The cost and latency difference is significant at scale.

Enable caching for read-heavy, stable data. Even a 60-second cache on a high-traffic endpoint can dramatically reduce backend load.

Keep Lambda functions warm for latency-sensitive paths. Cold starts add latency. Use Provisioned Concurrency for critical paths.

Set Lambda timeout slightly below the API Gateway timeout (e.g., 25 seconds vs the 29-second API Gateway max). This gives Lambda time to return a meaningful error rather than being cut off mid-execution.

Keep Lambda functions small and focused. Large dependency bundles slow cold starts. Use Lambda Layers for shared dependencies.

Use connection reuse in Lambda. Initialize database connections, SDK clients, and HTTP clients outside the handler function so they persist across warm invocations.

### Operational Excellence

Always use Infrastructure as Code (CloudFormation, SAM, CDK, Terraform). Manual console changes to APIs are not reproducible, auditable, or rollback-able.

Use separate AWS accounts or at minimum separate API Gateway APIs for dev/staging/prod. Stage variables can manage configuration differences, but the blast radius of a mistake in production should be isolated.

Set up CloudWatch Alarms on 5XXError, 4XXError, Latency P99, and throttling metrics. Be alerted on anomalies before customers notice.

Enable access logs and ship them to a centralized log analysis system. Access logs are the foundation of understanding API usage patterns, debugging issues, and detecting security anomalies.

Use X-Ray tracing to understand end-to-end request performance across your entire service graph.

Tag all API Gateway resources consistently (team, service, environment, cost center) for cost allocation and operational management.

---

## 22. Common Failure Modes and How to Think About Them

### 403 Forbidden

This always means something in the authorization layer rejected the request. Work through the layers:

Is the request missing an API key (`x-api-key` header) on a method that requires one? Is the API key valid and associated with a usage plan that includes this stage? Is the Lambda authorizer denying the request? Check CloudWatch logs for the authorizer Lambda. Is the IAM policy missing `execute-api:Invoke`? Is the resource policy blocking the source IP or VPC? Is CORS preflight failing (often manifests as 403 in browser)?

### 502 Bad Gateway

The backend returned a response that API Gateway could not process. For Lambda Proxy integration, this almost always means the Lambda function returned a response that doesn't match the expected structure (missing `statusCode`, non-string `body`, invalid JSON in the response). It can also mean Lambda threw an unhandled exception and returned an error object that API Gateway couldn't interpret.

Check Lambda CloudWatch logs first. Enable API Gateway execution logs to see exactly what the integration returned.

### 504 Gateway Timeout

The backend exceeded the integration timeout. The Lambda function (or HTTP endpoint) took longer than 29 seconds to respond. Solutions: optimize the backend performance, break the operation into async steps (Lambda → SQS → async processing), increase Lambda memory (more memory = more CPU = faster execution), or use a polling/webhook pattern for long-running operations.

### 429 Too Many Requests

Throttling. Either the account-level limit, stage-level limit, method-level limit, or usage plan limit was exceeded. Identify which limit is being hit via CloudWatch metrics. Implement exponential backoff with jitter in clients. Request limit increases if the traffic is legitimate. Use caching to reduce the number of requests that reach the integration.

### Latency Spikes

Investigate in layers using X-Ray. Is the spike in `IntegrationLatency` (backend problem) or in the difference between `Latency` and `IntegrationLatency` (API Gateway processing problem)? For backend latency, look at Lambda cold starts (provisioned concurrency, smaller package size), Lambda execution time (optimize code, increase memory), downstream service latency (DynamoDB hot partitions, RDS connection exhaustion). For API Gateway processing time, check if authorizer caching is disabled (every request invoking an authorizer Lambda adds latency).

### Inconsistent CORS Errors

CORS errors are often intermittent because preflight results are cached by the browser. After changing CORS configuration, the browser may still use the old cached preflight result. Test with curl (which ignores CORS) to separate actual API errors from browser CORS enforcement. Ensure both the OPTIONS method response AND the Lambda response include matching CORS headers. The `Access-Control-Allow-Origin` value must exactly match the request's `Origin` header — wildcard `*` does not work with credentials.

### Deployment Changes Not Taking Effect

Forgetting to redeploy is the most common cause of "I changed the API and it's not working." Changes to resources, methods, integrations, and authorizers in REST APIs are not live until deployed to a stage. HTTP APIs with auto-deploy enabled are the exception — they apply changes immediately. For REST APIs, always deploy after making changes.

---

_This guide reflects Amazon API Gateway capabilities as of mid-2024. AWS evolves services continuously — always cross-reference with the official AWS documentation for the latest feature availability and limits._
