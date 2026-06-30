# Distributed System Patterns & Building Blocks

Core architectural patterns referenced during system design. Use these as decision frameworks when selecting technologies.

---

## 1. Rate Limiting

Control request volume to protect services from abuse and overload:

| Algorithm | Mechanism | Trade-off |
|-----------|-----------|-----------|
| **Token Bucket** | Tokens refill at fixed rate; each request consumes a token | Allows controlled bursts up to bucket size |
| **Leaky Bucket** | Requests queue and drain at fixed rate | Smooth output, but no burst tolerance |
| **Fixed Window** | Count requests per time window (e.g., 100/minute) | Simple, but boundary spike problem (2× burst at window edges) |
| **Sliding Window Log** | Track exact timestamps of each request | Precise, but memory-intensive (stores every timestamp) |
| **Sliding Window Counter** | Weighted blend of current and previous window counts | Good precision/memory balance |

**Enforcement points:** API Gateway (global limits), Application layer (per-user/per-resource), or both.
**Response:** HTTP 429 with `Retry-After` header. Pair with client-side exponential backoff.

---

## 2. Caching Strategies

Select the caching pattern based on read/write ratio and staleness tolerance:

| Pattern | Mechanism | Best For |
|---------|-----------|----------|
| **Cache-Aside (Lazy)** | App checks cache → on miss, reads DB → writes result to cache | General read-heavy workloads |
| **Read-Through** | Cache itself fetches from DB on miss (transparent to app) | Simplified app logic, uniform cache layer |
| **Write-Through** | App writes to cache AND DB simultaneously | Strong consistency between cache and DB |
| **Write-Behind (Write-Back)** | App writes to cache only; cache flushes to DB asynchronously | Write-heavy workloads; risk of data loss on cache crash |
| **Refresh-Ahead** | Cache proactively refreshes entries before TTL expiry | Low-latency reads for predictable access patterns |

**Eviction policies** (when cache is full):
- **LRU (Least Recently Used):** Evicts the entry accessed longest ago. General-purpose default.
- **LFU (Least Frequently Used):** Evicts entries with the fewest accesses. Better for stable hot sets.
- **TTL (Time-To-Live):** Entries expire after a fixed duration regardless of access pattern.

**Invalidation strategies:** TTL-based (simple, eventual staleness), event-based (pub/sub on writes), or explicit delete-on-write.

**Cache Stampede / Thundering Herd:** When a hot key expires, hundreds of concurrent requests hit the DB simultaneously. Mitigate with:
- **Single-flight / request coalescing:** Only the first request fetches from DB; others wait for its result.
- **Probabilistic early expiration:** Each client randomly refreshes slightly before TTL expires.
- **Locking:** Distributed lock on the cache key during refresh.

---

## 3. Message Queue & Event Streaming Patterns

| Pattern | Examples | Semantics |
|---------|----------|-----------|
| **Point-to-Point** | SQS, RabbitMQ (default) | One consumer processes each message. Load distributed across consumers. |
| **Pub/Sub (Fan-out)** | SNS + SQS, Kafka topics, Google Pub/Sub | Each message delivered to ALL subscribers. Event broadcasting. |
| **Consumer Groups** | Kafka consumer groups | Partitions divided among group members. Ordered, parallel processing. |

**Delivery guarantees:**
- **At-most-once:** Fire and forget. Fast, but messages can be lost.
- **At-least-once:** Retry until acknowledged. May duplicate — consumers must be **idempotent**.
- **Exactly-once:** Requires transactional coordination (Kafka transactions). Highest cost.

**Dead Letter Queues (DLQ):** Messages that repeatedly fail processing are moved to a DLQ for manual inspection instead of blocking the main queue.

**Backpressure:** When consumers can't keep up, signal producers to slow down. Mechanisms: bounded queues (reject on full), flow control (TCP-level), or reactive streams.

---

## 4. Consistency Model Spectrum

CAP Theorem is a simplification. Real systems choose specific consistency models per data path:

| Model | Guarantee | Use Case |
|-------|-----------|----------|
| **Strong / Linearizable** | Every read returns the most recent write globally | Financial transactions, inventory counts |
| **Sequential** | All operations appear in a single total order (may lag real-time) | Distributed logs, event sourcing |
| **Causal** | Causally related operations are seen in order; concurrent ops may vary | Social feeds (replies after parent post) |
| **Read-Your-Writes** | A user always sees their own most recent write | User profile edits, shopping carts |
| **Eventual** | Reads may return stale data, but all replicas converge over time | DNS, CDN caches, analytics counters |

**Rule:** Default to eventual consistency. Upgrade to stronger models only for data paths where correctness is a business invariant.

**Quorum reads/writes:** In a replicated system with N replicas, require W writes and R reads where `W + R > N` to guarantee reading the latest write:
- **W=N, R=1:** Strong write consistency, fast reads. One slow node blocks all writes.
- **W=1, R=N:** Fast writes, strong read consistency. Read latency bounded by slowest replica.
- **W=⌈(N+1)/2⌉, R=⌈(N+1)/2⌉:** Balanced quorum. Tolerates minority failures for both reads and writes.

---

## 5. Data Partitioning & Sharding Strategies

| Strategy | Mechanism | Trade-off |
|----------|-----------|-----------|
| **Hash-Based** | `partition = hash(key) % N` | Uniform distribution, but no range queries and full reshuffle on node change |
| **Consistent Hashing** | Keys and nodes on a hash ring; key routes to next clockwise node | Minimizes data movement on scale-out (only K/N keys move). Use virtual nodes for balance. |
| **Range-Based** | Partition by key ranges (e.g., A-M, N-Z) | Supports range scans, but risks hotspots on popular ranges |
| **Directory-Based** | A central lookup service maps keys to shards | Maximum flexibility, but the directory is a SPOF and latency bottleneck |
| **Geographic** | Partition by user region (EU data on EU shards) | Satisfies data residency; high latency for cross-region queries |

**Rebalancing:** When shards become uneven (hot partitions), use **split-and-migrate** or **virtual shards** (pre-allocate many small virtual partitions that can be reassigned to physical nodes).

**Hot/Cold Storage Tiering:** Move infrequently accessed data to cheaper storage (S3 Glacier, HDD) while keeping hot data on fast storage (SSD, RAM). Reduces cost without sacrificing performance for active data.

---

## 6. Edge-to-Service Network Layering

The standard request path from client to backend:

```
Client → DNS (GeoDNS) → CDN (static assets) → L4/L7 Load Balancer → API Gateway → Service Mesh → Microservices → Database
```

| Layer | Role | Key Decision |
|-------|------|-------------|
| **DNS** | Resolves domain to IP. GeoDNS routes to nearest region. | TTL trade-off: low TTL = fast failover but more DNS queries |
| **CDN** | Caches static content at edge PoPs close to users | Pull-based (lazy) vs Push-based (pre-populate) |
| **Load Balancer (L4)** | Routes TCP connections by IP/port. No payload inspection. | Use for non-HTTP protocols (gRPC, WebSocket, DB proxying) |
| **Load Balancer (L7)** | Routes HTTP by URL, headers, cookies. Enables canary deploys. | Use for HTTP APIs; sticky sessions, header-based routing |
| **API Gateway** | Auth, rate limiting, request transformation, routing | Single entry point; enforce auth and rate limits here |
| **Service Mesh** | Sidecar proxies handle mTLS, retries, circuit breaking | Adds ~1ms per hop; centralizes cross-cutting concerns |

**Load Balancing algorithms:**
- **Round-Robin:** Requests distributed sequentially. Simple; ignores server capacity differences.
- **Weighted Round-Robin:** More traffic to higher-capacity servers.
- **Least Connections:** Routes to server with fewest active connections. Better for variable request durations.
- **IP Hash:** Hash client IP to select server. Pseudo-sticky sessions.
- **Consistent Hashing:** Used for cache-aware routing. Minimizes redistribution on node changes.

**Health Checks:**
- **Active:** LB periodically pings `/health`. Removes unhealthy nodes from rotation.
- **Passive:** LB monitors live traffic error rates. Marks nodes unhealthy after consecutive failures.

---

## 7. Database Selection Guide

Match data characteristics to the appropriate storage engine:

| Database Type | Examples | Best For | Limitations |
|---------------|----------|----------|-------------|
| **Relational (SQL)** | PostgreSQL, MySQL | Structured data, ACID transactions, complex joins | Vertical scaling ceiling; sharding is complex |
| **Key-Value** | Redis, DynamoDB | Simple lookups by key, sessions, caching | No complex queries; key-only access |
| **Document** | MongoDB, CouchDB | Semi-structured data, flexible schemas, nested objects | Weak joins; consistency trade-offs at scale |
| **Wide-Column** | Cassandra, ScyllaDB, HBase | High write throughput, time-series, IoT | No joins; query patterns must be designed upfront |
| **Graph** | Neo4j, Amazon Neptune | Relationship-heavy data (social graphs, recommendations) | Not suited for bulk analytical queries |
| **Search Engine** | Elasticsearch, Solr | Full-text search, log analytics, autocomplete | Not a primary store; eventual consistency |
| **Time-Series** | InfluxDB, TimescaleDB | Metrics, monitoring, sensor data | Append-only optimized; poor for random updates |
| **Object/Blob** | S3, GCS, Azure Blob | Large files (images, videos, backups) | No querying; access by key only |

**Decision rule:** Start with a relational database. Move to specialized stores only when a specific access pattern creates a measurable bottleneck.

**Normalization vs Denormalization:**
- **Normalized:** No duplicate data; updates in one place. Requires joins at read time — costly at scale.
- **Denormalized:** Pre-joined data stored together. Fast reads, but updates must propagate to all copies. Prefer for read-heavy paths where write frequency is low.

---

## 8. CQRS & Event Sourcing

**CQRS (Command Query Responsibility Segregation):** Separate the write model (commands) from the read model (queries). Each can be independently scaled, optimized, and stored differently.
- **Write side:** Normalized relational DB optimized for transactional integrity.
- **Read side:** Denormalized views (materialized views, Elasticsearch indexes) optimized for queries.
- **Sync mechanism:** Events propagated via message queue from write to read side. Introduces eventual consistency.
- **When to use:** Systems with vastly different read/write patterns (e.g., 100:1 ratio) or diverging read/write schemas.

**Event Sourcing:** Instead of storing current state, store an immutable append-only log of all state-changing events. Current state is reconstructed by replaying events.
- **Advantages:** Complete audit trail, temporal queries ("state at time T"), easy event-driven integration.
- **Challenges:** Event schema evolution, replay performance at scale (mitigate with snapshots), eventual consistency.

---

## 9. Real-Time Communication Patterns

| Protocol | Mechanism | Best For |
|----------|-----------|----------|
| **HTTP Long Polling** | Client holds request open; server responds when data available | Simple fallback; works through all firewalls |
| **Server-Sent Events (SSE)** | Server pushes over single HTTP connection (unidirectional) | Live feeds, notifications, dashboards |
| **WebSocket** | Full-duplex persistent TCP connection after HTTP upgrade | Chat, gaming, collaborative editing |

### HTTP Connection Pooling vs. WebSockets (WSS)
When designing network interfaces between services or client-to-server, evaluate connection pooling and WebSocket upgrades:
- **HTTP Connection Pooling:** Keeps a persistent pool of TCP connections open between a client (or service) and server. Avoids the high overhead of TCP/TLS handshake latency on every request.
  - *Trade-off:* Stateless. Best for standard request/response workloads, microservice REST/gRPC calls, and high-throughput query APIs. Server cannot push data directly without polling or SSE.
- **WebSockets (WSS):** Performs an HTTP upgrade to transition a connection to a bidirectional TCP stream.
  - *Trade-off:* Stateful. Best for bidirectional, low-latency, real-time communication (e.g. collaborative canvases, fast multiplayer gaming). Consumes permanent file descriptors and memory (~10-50 KB per socket) on the gateway. Harder to scale horizontally (requires sticky sessions, IP hash, or backend message bus coordination like Redis Pub/Sub to route messages between nodes).
- **Rule of Thumb:** Default to **HTTP Pooling** (combined with HTTP/2 or HTTP/3 multiplexing) for 95% of communication paths. Select **WebSockets (WSS)** only when bidirectional, real-time push is a strict functional requirement.

**Fan-Out Strategies:**
- **Fan-out-on-write (push):** On post, immediately write to all followers' feeds. Fast reads, but expensive writes for users with many followers (celebrity problem).
- **Fan-out-on-read (pull):** On feed open, query and merge posts from all followed users. Cheap writes, but slow reads.
- **Hybrid:** Push for normal users, pull for celebrities (>10k followers).

**Presence Tracking:** Track online/offline status with heartbeats. Clients send periodic heartbeat (e.g., every 30s) to a Presence Service backed by Redis with TTL. No heartbeat within TTL = offline.

---

## 10. Search & Indexing

**Inverted Index:** Maps each word/token to the list of documents containing it. Foundation of full-text search.
- **Tokenization:** Break text into terms (lowercase, stemming, stop-word removal).
- **Ranking:** TF-IDF, BM25, or learned ranking models to order results by relevance.

**Autocomplete / Typeahead:** Suggest completions as the user types.
- **Trie (prefix tree):** In-memory structure for fast prefix lookups. Pre-compute top-K suggestions per prefix.
- **Elasticsearch completion suggesters:** For larger datasets beyond memory capacity.

**Faceted Search:** Filter results by categories (price range, brand, color) using aggregation queries on indexed fields.

---

## 11. Deployment Strategies

| Strategy | Mechanism | Risk Level |
|----------|-----------|------------|
| **Rolling Update** | Gradually replace old instances with new | Low; brief version skew during rollout |
| **Blue-Green** | Two identical environments; switch traffic atomically from old to new | Zero-downtime; requires 2× infrastructure during deploy |
| **Canary Release** | Route small percentage (e.g., 5%) to new version; monitor before full rollout | Lowest risk; requires traffic splitting at LB |
| **Feature Flags** | Deploy code with features toggled off; enable per-user or per-region | Decouples deploy from release; adds code complexity |

**Containerization:** Package services as Docker containers for consistent builds across environments. Orchestrate with Kubernetes for auto-scaling, self-healing, and rolling deploys.

---

## 12. Distributed Systems Integration & Governance

When independent systems interact in a complex distributed architecture, establish boundaries and governance models:

### A. API Contracts
- **OpenAPI / Swagger:** Industry standard for RESTful interfaces. Facilitates documentation, mock servers, and automated SDK generation.
- **Protocol Buffers (Protobuf):** Binary serialization contract utilized by gRPC. Enforces type safety, strict schema definitions, and backward/forward compatibility rules (e.g. field numbering). Much smaller payload footprint than JSON.

### B. Capability & Service Discovery
Scale-out architectures dynamically allocate IP addresses to containers. Discovery systems map service names to physical IPs:
- **DNS-Based (e.g., Kubernetes CoreDNS):** Simple, standard, works out-of-the-box. Uses DNS SRV records. Cache expiration (TTL) must be managed to prevent routing to dead pods.
- **Key-Value Service Registries (Consul, etcd):** Stateful health monitoring. Nodes register their health status; services poll or subscribe to registries. Enables rich metadata tagging (e.g., routing to regional nodes or specific versions).

### C. Configuration-Driven Functionality
Decouple service logic changes from software deployment cycles:
- **Centralized Configuration (Consul, Spring Cloud Config, etcd):** Allows services to pull configuration settings (database credentials, system timeouts, threshold variables) at boot and subscribe to real-time configuration updates.
- **Feature Flags:** Allows gradual rollouts, runtime behavioral toggling, and circuit breaker bypass controls without code pushes or restarts.

### D. Shared Schemas vs. Interface Isolation
- **Shared Schemas (Schema Registry):** When using event-driven communication (e.g., Kafka with Avro), a Schema Registry enforces schema validation on write and read. Ensures publishers cannot write messages that break consumers.
- **Coupling Risks:** Directly sharing data models (e.g., database schema tables, shared source code packages, ORM entities) across service boundaries creates tight coupling. A schema modification in Service A forces redeployments of Service B.
- **Mitigation:** Enforce interface isolation. Services must exclusively expose data via contract-defined APIs (DTOs) and keep their underlying storage tables fully private.

---

## 13. Programming Language Selection Guide

When selecting programming languages, prioritize the option that best resolves the specific engineering requirements of the component, rather than choosing a default. Document the pros, cons, and why alternatives were rejected.

### A. Golang (Go)
- **Best For:** Microservices, network proxies, API Gateways, concurrency-intensive IO tasks.
- **Pros:**
  - Extremely fast compilation and quick startup times (great for serverless/containers).
  - Built-in lightweight concurrency primitives (goroutines and channels).
  - Simple syntax, high readability, short learning curve.
  - Low memory footprint compared to JVM.
- **Cons:**
  - Garbage collector (GC) overhead can cause minor latency spikes (not suitable for hard real-time systems).
  - Lacks advanced type system features compared to Rust (though generics are now supported).
- **Reject Reason Example:** Reject for high-throughput stream-processing engines where GC pause jitter violates a strict sub-millisecond P99.9 latency SLA.

### B. Rust
- **Best For:** CPU-intensive processing, latency-sensitive proxy layers, data engines, hardware integration.
- **Pros:**
  - Zero-cost abstractions and maximum performance (comparable to C/C++).
  - No Garbage Collector; compile-time borrow checker guarantees memory safety and thread safety.
  - Predictable latency profiles (no GC pause spikes).
  - Rich type system and excellent package manager (`cargo`).
- **Cons:**
  - Steep learning curve and slower developer velocity (borrow checker friction).
  - Significantly slower compilation times.
  - Smaller pool of experienced developers.
- **Reject Reason Example:** Reject for standard CRUD business APIs where developer velocity and rapid iteration are more critical than microsecond performance.

### C. Alternative Stacks (JVM vs. Node.js vs. Python)
- **Java/Kotlin (JVM):** Excellent for complex enterprise business logic and data integrations (e.g., Spring Boot, Apache Spark). *Cons:* High memory footprint, cold start latency.
- **Node.js (TypeScript):** Excellent for frontend-heavy orchestration APIs and fast prototyping. *Cons:* Single-threaded CPU performance bottlenecks.
- **Python:** Standard for Machine Learning and data science. *Cons:* Slow execution speed, dynamically typed.
