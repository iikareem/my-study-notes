# WebSocket Gateway: Study Notes

## 1. Core Idea

- A WebSocket is a long-lived TCP connection. The server must keep it open in memory, so the connection is **stateful**.
- REST requests are independent and stateless, so any server can handle any request.
- A WebSocket gateway holds only the raw connections (sockets, user IDs, subscriptions) and a routing map. Business logic lives in stateless services.
- Why split them: a code change to business logic does not drop every socket, so deploys stay cheap for users.

## 2. Sticky Sessions vs. Routing

- A load balancer only needs stickiness for the initial HTTP handshake. After the upgrade, the socket stays on that server.
- The real problem: when another service wants to send to user X, the system must know which gateway node holds user X's socket.
- Solutions: a shared registry (for example, a Redis key per user holding the gateway ID), or per-node channels.

## 3. Message Flow

- **Inbound:** Client → gateway → stream/topic → stateless service (business logic, DB write) → delivery path.
- **Outbound:** Service → delivery stream or Pub/Sub → gateway node holding the socket → client.
- The gateway forwards messages and does not interpret them.
- **Synchronous calls** from the gateway (auth, permission checks) are fine if bounded by timeouts and kept off the per-message hot path. Never query the database directly from the gateway.

## 4. Redis Pub/Sub vs. Redis Streams

- **Pub/Sub:** fire-and-forget. If no gateway is listening, the message is lost.
- **Streams:** persisted, support consumer groups and replay. Use them when a message must survive a gateway restart or a brief disconnect.
- **Two hops** (inbound stream → service → delivery stream) give durability, retries, commit-before-delivery, and multiple consumers.
- **One hop** (inbound stream → service → Pub/Sub) is simpler and fine when the client can always catch up from the database. The database is the source of truth; real-time push is best-effort.

## 5. Deployments

- Gateway deploys still drop sockets. Mitigate with graceful draining: stop accepting new connections, tell clients to reconnect, and move them gradually.
- Clients reconnect with exponential backoff and jitter to avoid a thundering herd.
- After reconnect, recover missed messages using sequence numbers or a last-event ID.

## 6. Authentication

- Check the token **during the HTTP handshake**, before the upgrade. Reject invalid requests so no WebSocket session or per-connection state is created.
- Correction: the TCP connection and its file descriptor already exist by then. Handshake auth avoids application-level memory, not the socket itself.
- Protect against unauthenticated floods with:
    - A handshake timeout
    - A cap on pending unauthenticated connections
    - Per-IP connection limits and rate limiting at the proxy
- Tokens can expire while a socket stays open. Options: close on expiry, require re-authentication over the socket, or check periodically.

## 7. Reverse Proxy / Load Balancer Layer

- Sits in front of the gateway and handles connection-level protection:
    - Per-IP connection and rate limits
    - Handshake and idle timeouts
    - TLS termination
    - Auth gating (NGINX `auth_request`, Envoy `ext_authz`, cloud LB integrations)
    - Connection draining during deploys
- Config essentials:
    - Forward the `Upgrade` and `Connection` headers
    - Set read/send timeouts longer than default, and pair with heartbeats
    - Ensure the proxy has enough file descriptors
- Limits: a proxy cannot stop volumetric DDoS. Use SYN cookies and edge DDoS protection for that.

## 8. File Descriptors

- On Linux, sockets, files, and pipes are all file descriptors (integer handles).
- Each WebSocket connection uses one descriptor, plus extra for Redis, DB, downstream services, and logs.
- The per-process cap is `RLIMIT_NOFILE` (`ulimit -n`), often 1024 by default. When hit, new connections fail with "too many open files."
- Configure:
    - Per-process limit: systemd `LimitNOFILE=`, or `/etc/security/limits.conf`. Check container runtime limits too.
    - System-wide limit: `fs.file-max`.
    - Monitor open descriptor count and alert before the limit.
    - Use connection pooling to reduce outbound descriptors.
- Descriptors are one ceiling among several: memory per connection, CPU for heartbeats and messages, and network capacity. Load-test to find the real number.

## 9. Backpressure for Slow Clients

- Problem: a slow client makes messages queue up in server memory. Many slow clients exhaust RAM.
- Solutions:
    - Cap the per-connection send buffer. When full, drop the oldest messages or disconnect.
    - Monitor drain rate and close stalled sockets.
- Disconnecting a slow client is usually better than letting it hurt everyone else. It can reconnect and catch up.

## 10. Ordering Guarantees

- TCP keeps order within one connection. Order breaks across multiple hops, retries, or multiple gateway nodes.
- Solutions:
    - Attach a sequence number per conversation or user so the client can detect gaps and reorder.
    - Route all messages for one conversation through the same partition or stream. Hash the conversation ID to pick it.
        - Kafka: one partition per key; order guaranteed within a partition.
        - Redis Streams: one stream per conversation, or a fixed set chosen by hash.
- Skip this if order doesn't matter (for example, latest-value-only data like live prices).
- Partitioning fixes order at the source; sequence numbers protect what the client actually receives.