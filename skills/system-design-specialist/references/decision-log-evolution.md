# Architecture Evolution & Decision Logs (ADRs)

A guide to creating professional decision logs and planning evolutionary roadmaps for growing systems.

---

## 1. The Architecture Evolution Roadmap

A professional Principal Engineer knows that building for 100 million users on day one is a recipes for failure (wasted capital, complexity, delayed launch). Instead, design for **progressive scaling**.

Map the system design across three distinct stages:

### Design 0: Minimum Viable System (Day 1)
- **Target Scale:** 1,000 to 10,000 DAU.
- **Goal:** Speed to market, simplicity, minimal operational overhead.
- **Pattern:** Monolith architecture, single Relational Database (RDS), local file storage, basic caching (in-memory or single Redis node).
- **Bottlenecks:** Single point of failure (SPOF) on database, CPU limits on single monolith instance.

### Design 1: Scaled & Separated System (Year 1)
- **Target Scale:** 100,000 to 1 million DAU (10x-100x scale).
- **Goal:** High Availability, component separation, database relief.
- **Pattern:** Microservices or separate service modules, database read replicas, distributed cache layer (Redis Cluster), CDN for assets, introduction of message queues (RabbitMQ/SQS) to handle async processing.
- **Bottlenecks:** Monolithic database writes, service communication latency, state consistency across systems.

### Design 2: Planet-Scale Distributed System (Year 3+)
- **Target Scale:** 10 million+ DAU (1000x scale).
- **Goal:** Regional isolation, horizontal write scalability, infinite scaling capability.
- **Pattern:** Database horizontal sharding (hash-based/consistent hashing) or NoSQL migration, multi-region active-active deployment, globally distributed content networks, event-sourced or message-streaming architecture (Kafka/Kinesis), localized data storage.

---

## 2. Decision Log (Architecture Decision Records - ADR)

Decisions must be recorded with context. A Decision Log or ADR ensures that choices are justified, alternatives are assessed, and trade-offs are accepted.

### ADR Format template

```markdown
### ADR-00X: [Title of Decision]

- **Status:** [Proposed / Accepted / Rejected]
- **Context:** [Describe the problem we are solving and the constraints.]
- **Alternatives Considered:**
  1. **Option A:** [Description]
     - *Pros:* ...
     - *Cons:* ...
  2. **Option B:** [Description]
     - *Pros:* ...
     - *Cons:* ...
- **Decision:** [Which option did we choose and why?]
- **Trade-offs / Consequences:** [What are the downsides we are accepting? What are the implications for other components?]
```

### Worked Example 1: Live WebSocket vs. HTTP Connection Pooling / Long Polling
- **Status:** Accepted.
- **Context:** We need to choose the communication interface between the client app and the notification gateway. The gateway must deliver real-time chat alerts with less than 200ms latency at high frequency.
- **Alternatives Considered:**
  1. **HTTP/2 Connection Pooling & Long Polling:**
     - *Pros:* Stateless endpoints, easier to load balance, reuses connections, works out of the box with existing API gateway rules.
     - *Cons:* Server cannot push messages unilaterally without client initiating a poll request first, introducing latency overhead.
  2. **WebSocket (WSS - Stateful Persistent Connection):**
     - *Pros:* True full-duplex bidirectional messaging, extremely low overhead after handshake, allows instant server push.
     - *Cons:* Stateful, consumes file descriptors and memory per connection on gateway nodes, requires sticky load balancing and a presence heartbeat checker.
- **Decision:** WebSocket (WSS).
- **Consequences:** We must implement a stateful WebSocket gateway layer using an AWS Application Load Balancer with sticky sessions, and build a dedicated Presence Service backed by a Redis cluster to track active connections.
- **Why Others Were Rejected:** HTTP connection pooling was rejected because the server must push notifications immediately to the client without waiting for the client to poll.

### Worked Example 2: Service Stack Programming Language Selection
- **Status:** Accepted.
- **Context:** We need to design the high-throughput matching engine service. The service must process location coordinate updates from 500,000 drivers and match them to riders under a strict P99 latency SLA of 50ms.
- **Alternatives Considered:**
  1. **Golang (Go):**
     - *Pros:* Built-in lightweight concurrency primitives (goroutines), fast startup, low memory footprint, simple code maintenance, rapid developer velocity.
     - *Cons:* Garbage collector pauses can introduce latency jitter (1-10ms P99 spikes).
  2. **Rust:**
     - *Pros:* Predictable sub-millisecond execution latency with no garbage collection pauses, memory safety guarantees, compiler enforcement of thread-safety.
     - *Cons:* Higher syntax complexity, slower developer iteration speed, longer compilation cycles.
  3. **Java (JVM / Spring Boot):**
     - *Pros:* Rich ecosystem, mature thread pool models, great developer availability.
     - *Cons:* Heavy RAM footprint (~1-2 GB baseline vs Go's ~30 MB), significant Garbage Collection latency spikes (GC pauses of 50ms+ under heavy load), JVM warm-up period.
- **Decision:** Golang.
- **Consequences:** Go is selected because the 50ms latency SLA is accommodating enough to handle Go's minor GC jitter (~1-2ms), while Go's goroutines allow simple and readable matching logic. We accept the need to tune Go's GC settings (e.g. `GOGC` or `GOMEMLIMIT`) under peak QPS.
- **Why Others Were Rejected:** Rust was rejected because developer velocity and time-to-market are critical for this project phase, and the performance benefit of Rust does not justify the slow compilation and steeper learning curve. Java was rejected because GC pauses can exceed the total 50ms latency budget, and JVM's memory footprint would significantly increase operational VM costs.

---

## 3. Core SRE & Design Ledgers

Principal Engineers maintain structured registers to align requirements, track unknowns, and justify architectural decisions:

### A. Assumption Ledger
Track and continuously re-verify assumptions to keep the design grounded:

| Assumption ID | Assumption Description | Impact Sensitivity (High/Med/Low) | Confidence Level (High/Med/Low) | Verification Method / Source |
|---------------|------------------------|----------------------------------|--------------------------------|------------------------------|
| **ASM-001** | Peak write traffic is 10k QPS | High | Low | Log analysis / Product PM |
| **ASM-002** | Average payload object is 500 bytes | Med | High | Schema size calculation |
| **ASM-003** | 80% of traffic is concentrated in EU | High | Med | GeoIP analytics data |

### B. Risk Ledger
Identify failure vectors, likelihoods, and code-level mitigations before drawing the final architecture:

| Risk ID | Risk Description | Likelihood (High/Med/Low) | Impact (High/Med/Low) | Actionable Mitigation Plan |
|---------|------------------|---------------------------|-----------------------|----------------------------|
| **RSK-001** | Hot database partition (celebrity write hotspot) | Med | High | Partition users by `hash(user_id) + salt` |
| **RSK-002** | Cascading thread starvation under API gateway failure | High | High | Circuit breakers (resilience4j) & strict timeouts |
| **RSK-003** | Data inconsistency between EU and US nodes | Low | High | Eventual consistency validation daemon |

### C. Complexity Budget Ledger
Enforce architectural discipline by forcing every service, queue, cache, or database node to justify its existence:

| Component Name | Specific Problem Solved | Added Operational Cost | Feasible Simpler Alternative | Rejection Reason for Alternative |
|----------------|-------------------------|------------------------|------------------------------|----------------------------------|
| **Kafka Cluster** | Decouples write spikes, enables multi-consumer fanout | High infrastructure & Zookeeper/KRaft complexity | AWS SQS | Retaining offset order is mandatory; SQS doesn't support consumer groups at scale |
| **Redis Cache** | Saves database read CPU for hot user profiles | Cache invalidation bugs, TTL synchronization | DB Index Cache | Database query cache suffers from write-invalidation thrashing |
