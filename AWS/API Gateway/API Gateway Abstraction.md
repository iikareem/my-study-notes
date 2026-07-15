---
tags:
  - api-gateway
  - aws
  - certification
---

**Hub:** [[AWS MOC]] · **Role:** Extra
**Also:** [[API Gateway]]

# API Gateway — Hierarchical Abstraction

---

## 1. What It Is

> **Fully managed reverse proxy** that sits between clients and backends. Not an app server — a programmable front door.

```
Client ──▶ API Gateway ──▶ Backend (Lambda / HTTP / AWS Service)
              │
              ├── Authenticate & Authorize
              ├── Validate & Transform
              ├── Route & Rate-limit
              └── Log & Monitor
```

---

## 2. Three API Flavors (the top-level split)

```
                      ┌──────────────────────────────────────────────────────────────┐
                      │                    API GATEWAY                              │
                      ├────────────┬────────────────────┬───────────────────────────┤
                      │   REST     │      HTTP          │        WebSocket          │
                      │   (v1)     │      (v2)          │                           │
                      ├────────────┼────────────────────┼───────────────────────────┤
                      │ Full-feat. │ Simple & cheap     │ Persistent bidirectional   │
                      │ VTL, cache │ Lambda/HTTP proxy  │ Real-time comm            │
                      │ Usage plns │ JWT native auth    │ Pub/sub, chat, live dash  │
                      │ AWS direct │ No VTL, no cache   │ Connection IDs, Mgmt API  │
                      └────────────┴────────────────────┴───────────────────────────┘
```

**Decision:**
- Need VTL/caching/usage plans? → **REST**
- Simple Lambda/HTTP proxy, low cost? → **HTTP**
- Real-time bidirectional? → **WebSocket**

---

## 3. Core Concepts Hierarchy

```
API (top-level container, unique ID)
│
├── Resources (URL path tree: /users/{userId})
│   └── Methods (GET, POST, PUT, DELETE, ANY)
│       ├── Method Request (auth, validation, API key check)
│       ├── Integration Request (route to backend, VTL transform)
│       ├── Integration Response (backend → client transform)
│       └── Method Response (final shape returned to client)
│
├── Stages (named snapshots: dev, staging, prod)
│   ├── Stage Variables (env vars per stage)
│   ├── Canary Releases (% traffic to new deployment)
│   └── Per-stage: throttling, caching, logging
│
├── Deployments (immutable snapshots of API config)
│
├── Authorizers (IAM / Cognito / Lambda)
│
├── Usage Plans + API Keys (per-consumer throttling & quotas)
│
└── Custom Domains + Base Path Mappings (api.mycompany.com/v1 → API+stage)
```

---

## 4. Endpoint Types (how the API is exposed)

```
                 ┌─────────────────────┐
                 │   ENDPOINT TYPE     │
                 ├─────────┬───────────┼─────────────┐
                 │         │           │             │
             REGIONAL  EDGE-OPT.    PRIVATE        │
             (single   (CloudFront  (VPC only)     │
              region)   global)                     │
                                                    │
              ┌─────────────────────────────────────┘
              │
         Can also put behind your own CloudFront for custom behavior
```

---

## 5. Integration Types (how it connects to backends)

```
                    ┌────────────────────────────┐
                    │      INTEGRATION TYPE      │
                    ├──────────┬─────────┬───────┤
                    │          │         │       │
             AWS_PROXY    AWS      HTTP_PROXY  HTTP (non-proxy)
              (Lambda     (Lambda  (passthru  (VTL transform
               passthru)   non-proxy,  to any    on HTTP
                           DynamoDB,   HTTP      backend)
                           SQS, etc.)   backend)
                    │          │         │       │
                    └──────────┴─────────┴───────┘
                                        │
                                     MOCK (static response, no backend)
```

---

## 6. Request Lifecycle (the 4 phases)

```
   CLIENT
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: METHOD REQUEST                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Authorizer runs (IAM/Cognito/Lambda)                 │   │
│  │  • Request validation (JSON Schema models)               │   │
│  │  • API key check                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 2: INTEGRATION REQUEST                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • VTL mapping template (non-proxy)                     │   │
│  │  • Passthrough (proxy)                                  │   │
│  │  • Route to Lambda / HTTP / AWS service                 │   │
│  └──────────────┬──────────────────────────────────────────┘   │
└─────────────────┼──────────────────────────────────────────────┘
                  │
                  ▼
              BACKEND  ◄── Integration timeout: 29s max
                  │
                  ▼
┌─────────────────┼──────────────────────────────────────────────┐
│  PHASE 3: INTEGRATION RESPONSE                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • VTL mapping template (non-proxy)                     │   │
│  │  • Status code mapping                                  │   │
│  │  • Header transformation                                │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  PHASE 4: METHOD RESPONSE                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Status codes declared                                │   │
│  │  • Response headers + body models                       │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
     │
     ▼
   CLIENT
```

---

## 7. Security Layers (applied in order)

```
Request ──▶ 1. Resource Policy  (IP allow/deny, VPC, cross-account)
                │
                ▼
            2. API Key Check  (if required)
                │
                ▼
            3. Authorizer  (IAM / Cognito / Lambda)
                │
                ▼
            4. WAF  (SQL injection, XSS, IP reputation, geo-block)
                │
                ▼
            5. Backend  (your own auth logic)
```

### Authorizer types:

```
AUTHORIZER
├── IAM (AWS_IAM)
│   └── SigV4 signing, IAM policies on execute-api:Invoke
│   └── Best for: service-to-service within AWS
│
├── Cognito User Pool
│   └── Validates JWT (sig, exp, aud, iss) — no Lambda needed
│   └── Best for: user-facing apps with Cognito auth
│
└── Lambda Authorizer (custom)
    ├── Token-based: validates a single token
    ├── Request-based: full request context available
    └── Best for: custom auth logic, 3rd-party identity providers
```

---

## 8. Throttling Hierarchy (applied bottom-up, min wins)

```
                    ┌─────────────────────────┐
                    │   Account-Level Limit   │  ← Hard ceiling per region
                    │   Default: 10,000 RPS   │      shared across ALL APIs
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Stage-Level Limit     │
                    │   Default rate + burst   │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Method-Level Limit    │
                    │   Per-route override    │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Usage Plan Limit      │
                    │   Per API key + quota   │
                    └───────────┬─────────────┘
                                │
                    ┌───────────▼─────────────┐
                    │   Effective Limit =     │
                    │   MIN of all above      │
                    └─────────────────────────┘
```

---

## 9. Caching Summary

| Property | Details |
|----------|---------|
| **Scope** | Stage-level, per-method override |
| **TTL** | 0–3600s (default 300s) |
| **Cache key** | Full path + designated query params + headers |
| **Invalidation** | `Cache-Control: max-age=0` (may require IAM) |
| **Encryption** | At-rest option |
| **Available?** | REST only — NOT for HTTP or WebSocket |

---

## 10. VTL — When You Need It

> Velocity Template Language — transforms request/response payloads. **REST APIs only.**

```
Client body ──▶ VTL template ──▶ Backend format
    {"name":"K"}      ↓       {"Item":{"name":{"S":"K"}}}
```

Use cases:
- DynamoDB direct integration (construct the API call in VTL)
- Reshape Lambda non-proxy input/output
- Strip/map headers between client and backend

---

## 11. Stages & Deployments — Key Mental Model

```
┌─────────────────────────────────────────────────────┐
│  API (draft — changes not live!)                    │
│  ┌──────┐  ┌──────┐  ┌──────┐                      │
│  │ v1.0 │  │ v1.1 │  │ v2.0 │  ← Deployments      │
│  └──┬───┘  └──┬───┘  └──┬───┘   (immutable snaps)  │
│     │         │         │                           │
│     ▼         ▼         ▼                           │
│  ┌──────┐  ┌──────┐  ┌──────┐                      │
│  │ dev  │  │staging│  │ prod │  ← Stages           │
│  └──────┘  └──────┘  └──────┘                      │
│              │   │                                   │
│          base│   │canary (x% traffic)               │
└─────────────────────────────────────────────────────┘
```

**Golden rule:** In REST APIs, changes are NOT live until deployed to a stage.

---

## 12. Architecture Patterns

```
SERVERLESS API              MICROSERVICES GATEWAY
┌──────────┐                ┌──────────┐
│  Client  │                │  Client  │
└────┬─────┘                └────┬─────┘
     ▼                           ▼
┌──────────┐               ┌──────────┐
│  API GW  │               │  API GW  │
└────┬─────┘               └──┬───────┘
     ▼                        ├────────┬────────┬───────┐
┌──────────┐                  ▼        ▼        ▼       ▼
│  Lambda  │               /users  /orders  /payments  /inv
└────┬─────┘                  │        │        │       │
     ▼                   Lambda    ECS      EC2    Lambda
┌──────────┐
│ DynamoDB │
└──────────┘

BFF (Backend for Frontend)    DIRECT AWS INTEGRATION
┌──────────┐  ┌──────────┐   ┌──────────┐
│ Mobile   │  │  Web     │   │  Client  │
│ API GW   │  │  API GW  │   └────┬─────┘
└────┬─────┘  └────┬─────┘        ▼
     │              │         ┌──────────┐
     │  different   │         │  API GW  │
     │  response    │         └────┬─────┘
     │  shapes      │              ▼
     ▼              ▼         DynamoDB PutItem
  Lambdas        Lambdas     (VTL constructs the call)
```

---

## 13. Monitoring — Key Metrics

```
METRIC              WHAT IT TELLS YOU
─────────────────────────────────────────────────
Count               Traffic volume
Latency             Total end-to-end (client POV)
IntegrationLatency  Backend performance
4XXError            Client errors / auth issues
5XXError            Server errors / crashes
CacheHitCount       Caching effectiveness
CacheMissCount      Cache miss rate
```

**Tools:** CloudWatch Logs (execution + access), X-Ray (distributed traces)

---

## 14. Critical Limits

| Item | Limit |
|------|-------|
| Max payload | 10 MB |
| Integration timeout | 29s (REST/WS), 30s (HTTP) |
| Max RPS (account, default) | 10,000 |
| Burst (account, default) | 5,000 |

---

## 15. Exam-Critical Decision Tree

```
What kind of API?
├── Real-time, persistent connection? ──▶ WebSocket
├── Need caching, VTL, usage plans? ──▶ REST (v1)
├── Simple Lambda/HTTP proxy, cheap? ──▶ HTTP (v2)

How to authorize?
├── SigV4, service-to-service? ──▶ IAM
├── JWT/Cognito, simple? ──▶ Cognito (or JWT native on HTTP)
├── Custom logic, 3rd-party IdP? ──▶ Lambda Authorizer

How to control traffic?
├── Per-consumer limits? ──▶ Usage Plans + API Keys
├── Per-stage/method limits? ──▶ Stage/Method throttling

How to expose the API?
├── Global clients, low latency? ──▶ Edge-optimized
├── Same region clients? ──▶ Regional
├── VPC-only, internal? ──▶ Private + VPC Endpoint

Integration type?
├── Lambda with full event? ──▶ AWS_PROXY
├── AWS service directly? ──▶ AWS (VTL)
├── External HTTP endpoint? ──▶ HTTP_PROXY
├── Static response / mock? ──▶ MOCK
```

---

## 16. One-Sentence Abstraction

> API Gateway is a **programmable, serverless reverse proxy** that secures, transforms, routes, and monitors HTTP/WebSocket traffic between clients and backends — with three distinct modes (REST / HTTP / WebSocket) chosen by feature need.
