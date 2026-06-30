# The 18-Step Progressive Refinement Framework

A systematic guide for executing a thorough, professional, and non-abstract system design. 

---

## Phase 1: Problem Definition & Constraints (Steps 1-7)

### Step 1: Problem Framing
- **Goal:** Define the high-level mission and the target domain. Identify the core user needs.
- **Guideline:** Write a single sentence defining the system's purpose. Avoid technical jargon or premature solutions.
- **Example:** "Design a globally available ride-sharing matching platform that connects drivers and riders in real-time."

### Step 2: Clarifying Questions & Assumptions
- **Goal:** Uncover hidden assumptions and reduce ambiguity. Establish the core scope of the session.
- **Guideline:** List 5-10 targeted questions for the user regarding user roles, scale, scope, and requirements. **Do not move forward without explicit user confirmation of these assumptions.**
- **Key Checkpoints:**
  - Who are the users?
  - What are the write-to-read ratios?
  - Are custom features (e.g., scheduled rides, analytics) in scope?
  - Are we building for a single country or global?
  - **Availability & Redundancy Targets:** What are the targets for high availability (e.g., 99.9% vs 99.999% uptime), redundancy levels (e.g., active-active multi-region, active-passive hot standby), and recovery bounds (RTO/RPO limits)?
  - **Replication Tolerances:** Is data replication required across zones or regions? What are the tolerances for replica lag, temporary read staleness, or eventual consistency under partitions?

### Step 3: Functional & Non-Functional Requirements & Invariants
- **Goal:** Formalize what the system *must do* (functional), how it *must perform* (non-functional), and what *must never happen* (invariants).
- **Functional Requirements:** (Limit to 3-5 core features)
  - Users can request a ride.
  - Nearby drivers receive notification of requests.
  - Driver can accept/reject.
- **Non-Functional Requirements & Budgets:** (Quantifiable targets)
  - **Latency Budget (P99 = 200ms)**: Allocate total latency budget across components (e.g., DNS/TLS: 30ms, Gateway: 10ms, Auth: 15ms, API Logic: 35ms, Database: 50ms, Serialization: 15ms, Safety Buffer: 45ms).
  - **Error Budget (99.99% Availability)**: Calculate operational downtime allowance (52.6 minutes/year or 4.38 minutes/month) to drive replication and redundancy decisions.
  - **Consistency**: Eventual consistency for history; strong consistency for active ride state/payment.
- **System Invariants (Business Correctness)**:
  - Define invariants that *cannot happen* under any circumstances (e.g., "Money disappears from user balance", "Ride matching duplicates", "Double credit card charge", "Data lost on database crash").

### Step 4: Workload Characterization
- **Goal:** Define traffic patterns, load profiles, and read/write characteristics.
- **Guideline:** Categorize the workload characteristics before choosing technologies. Use this table to map workload characteristics to system properties:

| Workload Characteristic | Architectural Impact | Causal Design Decision |
|-------------------------|----------------------|------------------------|
| **Read-heavy?** | Dictates caching strategy | Cache-aside (Redis) / CDN at the edge |
| **Write-heavy?** | Dictates storage engine choice | Log-structured engines (LSM/Cassandra) over B-Trees |
| **Large objects? (Photos/Videos)** | Dictates storage medium | Object Store (S3) with CDN instead of DB blobs |
| **Small objects? (Metadata)** | Dictates index requirements | Key-value store or Relational DB with proper indexes |
| **Sequential writes?** | Log-structured writing | Append-only files / Write-ahead logging |
| **Random access reads?** | Index layout complexity | Memory-cached B-Tree / hash-index lookups |
| **Bursty traffic spikes?** | Requires buffer protection | Message Queue (Kafka/SQS) and Auto-scaling rules |
| **Global users?** | Requires physical separation | GeoDNS routing / Multi-region replication fences |

### Step 5: Back-of-the-Envelope Calculations
- **Goal:** Establish baseline mathematical numbers for QPS, bandwidth, storage, and caching.
- **Guideline:** Calculate Read/Write QPS (average and peak), data ingress/egress bandwidth, storage growth (daily/monthly/yearly with 30% overhead), and working set cache size (80/20 rule). Refer to [estimation-guide.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/estimation-guide.md) §3 for formulas.

### Step 6: Capacity Planning
- **Goal:** Translate back-of-the-envelope numbers into physical hardware requirements (server count, cache nodes, DB shards).
- **Guideline:** Use hardware capacity limits from [estimation-guide.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/estimation-guide.md) §2 to size infrastructure.

### Step 7: Constraints & Trade-offs
- **Goal:** Identify initial engineering limits (e.g., CAP Theorem, budget, hardware).
- **Guideline:** Choose between Consistency and Availability under partition (CP vs AP). Document why this choice was made.
- **Decision Documentation:** Every choice made in this step must produce a **Decision Record Table** (see SKILL.md “Comprehensive Decision Documentation”). This includes but is not limited to: consistency model (CP vs AP), budget constraints leading to technology exclusions, and any hardware or cloud platform selection.

---

> [!IMPORTANT]
> **PHASE 1 GATE (UNDERSTAND):**
> *Is the problem sufficiently understood (scale quantified, workload characterized, invariants established, and assumptions aligned) to justify drawing an architecture?* **Pause and confirm with the user before proceeding to Phase 2.**

---

## Phase 2: Architectural Design (Steps 8-10)

### Step 8: High-Level Architecture
- **Goal:** Draw the overall architecture skeleton.
- **Guideline:** Present a clear component diagram showing:
  - Clients (Mobile/Web).
  - Edge/Routing (DNS, CDN, API Gateway, Load Balancer).
  - Business Logic Services (Stateless microservices).
  - Storage Layer (Databases, Caches, Blobs, Queues).
  - Directions and protocols of data flow (HTTPS, gRPC, WebSockets, AMQP).

### Step 9: Data Model & APIs
- **Goal:** Define how data is represented and how services communicate.
- **Guideline:**
  - **API Contract:** Write REST, GraphQL, or gRPC specifications for main endpoints (request, response, error codes). Establish explicit contracts (OpenAPI schema or Protobuf schemas).
  - **Connection Protocol:** Explicitly evaluate HTTP connection pooling vs WebSockets (WSS). Document connection lifecycle, resource overhead (memory/file descriptors), and statefulness trade-offs.
  - **Integration & Governance:** Detail capability discovery (service discovery via DNS, Consul, or etcd) and configuration-driven functionality (dynamic configurations, feature flags, routing rules). Address schema sharing (e.g., Avro Schema Registry, shared libraries, shared database schemas) vs. complete interface isolation, highlighting coupling risks.
  - **Pagination:** Specify cursor-based (for real-time feeds, infinite scroll) vs offset-based (for static listings). Cursor-based preferred at scale (stable under concurrent writes).
  - **Database Schema:** Provide tables/collections structure, indexes, and primary/foreign/partition keys.
  - **Storage Selection:** Justify SQL vs NoSQL (Document, Wide-column, Key-value). Refer to [patterns.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/patterns.md) §7 for database selection guide.
- **Decision Documentation:** Every choice made in this step must produce a **Decision Record Table**. This includes but is not limited to: API protocol (including HTTP pooling vs WSS justification), API contracts, pagination strategy, database engine and type, storage medium (block vs object vs blob vs file), data partitioning approach (hash-based vs range-based sharding), index type (B-tree vs LSM vs hash), serialization format, and any data structure selection.

### Step 10: Component Responsibilities & Complexity Budget
- **Goal:** Clearly define what each block in your high-level architecture is responsible for.
- **Complexity Budget Guideline:** For every component, service, queue, cache, or database node added to the architecture, ask:
  - *Why is this component needed?*
  - *What specific problem does it solve?*
  - *What operational burden (maintenance, failure mode, monitoring) does it introduce?*
  - *Can we remove it or replace it with a simpler alternative (e.g., database lookup instead of Redis cache)?*
- **Programming Language & Stack Selection:** Choose the programming language (e.g., Golang vs Rust vs others) best suited to resolve the component's problem (latency constraints, developer velocity, library ecosystem). You must document the choice in a Decision Record Table, outlining pros/cons and explaining exactly why other major languages were not selected.
- **Example:** "Geohash Service (Golang): Low-memory stateless service written in Go (chosen over Rust for faster development cycles and built-in concurrency controls; chosen over Java/Node.js to avoid GC pauses and heavy memory footprint). Holds ephemeral driver location indexes in memory using Redis Geospatial index. Responsible for answering 'which drivers are within 5km of coordinate X'."

---

> [!IMPORTANT]
> **PHASE 2 GATE (DESIGN):**
> *Can every single component in the architecture justify its existence against the Complexity Budget, and is every technology choice traced directly to a requirement?* **Pause and confirm with the user before proceeding to Phase 3.**

---

## Phase 3: Operational Stability & Refinement (Steps 11-18)

### Step 11: Reliability & Failure Analysis
- **Goal:** Design for the "hostile path" — how the system behaves during outages, partitions, and cascading failures.
- **Guideline:** 
  - **Fault Tolerance & SPOF Elimination:** Actively audit the system to ensure no single point of failure (SPOF) exists. Plan redundancy (N+1, 2N, active-active vs active-passive) and automatic failover mechanisms (load balancer health check intervals, automated DNS failover, heartbeat timeouts) for web servers, databases, queue clusters, and routing layers.
  - **Replication Strategy & Impact:** Formulate replication strategy for database and state layers (single-leader, multi-leader, leaderless; synchronous vs asynchronous). Analyze and document replication impacts: replica lag (leading to read-after-write consistency issues), network bandwidth overhead, write amplification, and conflict resolution (e.g. LWW, CRDTs) under partitions.
  - **Outage Prevention:** Refer to [production-engineering.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/production-engineering.md) §1 for cascading outage prevention, DR (RPO/RTO), multi-region active-active, retries/backoff/jitter, and distributed transactions.

### Step 12: Security & Compliance
- **Goal:** Secure data at rest, in transit, and in memory. Define authorization boundaries and regulatory compliance.
- **Guideline:** Refer to [production-engineering.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/production-engineering.md) §4 for RBAC vs ABAC, data residency, JWT vs session auth, and TLS/mTLS patterns.
- **Decision Documentation:** Every choice made in this step must produce a **Decision Record Table**. This includes but is not limited to: authentication mechanism, authorization model, transport security, encryption at rest approach, WAF selection, secrets management tool, certificate management strategy, and data residency approach.

### Step 13: Observability & SLOs
- **Goal:** Establish how you will monitor, alert on, and measure the health of the system.
- **Guideline:**
  - **Observability-Driven Development (ODD):** Design telemetry alongside application logic. Instrument every request-path, database call, and message queue publish with contextual tags. Define required metrics, logs, and spans during the API contract definition phase.
  - **Distributed Tracing & Context Propagation:** Establish a trace propagation model (e.g., W3C Trace Context). Detail how the same Trace ID is preserved across all network boundaries (HTTP requests, gRPC metadata) and asynchronous boundaries (e.g. extracting trace context from Kafka/RabbitMQ message headers on consume to link the consumer span to the publisher span).
  - **Core Telemetry Metrics:**
    - *RED Method (for Services):* Rate (QPS), Errors (5xx rate/timeouts), Duration (P50/P90/P99 latency distribution).
    - *USE Method (for Infrastructure):* Utilization (% CPU/Memory/Disk busy), Saturation (queue depth, connection pool wait times), Errors (hardware warnings, socket errors).
    - *Business-Level Metrics:* Measure transactional state (e.g. payment success rate, active driver/passenger matches, search conversion rate).
  - **Logs & Spans Integration:** Standardize on structured JSON logs containing `trace_id` and `span_id`. Embed business keys (e.g., `user_id`, `tenant_id`, `payment_method`) as trace attributes/baggage to enable high-cardinality analysis.
  - Refer to [production-engineering.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/production-engineering.md) §2 for detailed observability implementation patterns and §3 for SLI/SLO/SLA formulation.
- **Decision Documentation:** Every choice made in this step must produce a **Decision Record Table**. This includes logging aggregation framework, metrics collection engine, distributed tracing tool, context propagation header format (W3C vs B3), and trace sampling strategy (e.g. constant vs probabilistic).

### Step 14: Cost Analysis
- **Goal:** Estimate the operational cost of the architecture.
- **Guideline:** Calculate the cost of server resources, storage, caching, network egress, CDNs, and third-party APIs. Compare alternatives to optimize ROI.

### Step 15: Architecture Evolution Roadmap
- **Goal:** Plan how the system grows from MVP to planet-scale over time.
- **Guideline:** Refer to [decision-log-evolution.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/decision-log-evolution.md) §1 for the Design 0 → Design 1 → Design 2 progressive scaling framework.

### Step 16: Decision Log with Alternatives (Completeness Audit)
- **Goal:** Produce a consolidated, numbered Architecture Decision Log and verify that *every single choice* in the design is documented.
- **Guideline:**
  1. Walk through **every component** in the architecture diagram: each service, database, queue, cache, CDN, WAF, load balancer, DNS layer, storage system, and external dependency.
  2. Walk through **every algorithm and data structure**: sharding strategy, index type, hashing algorithm, sorting approach, rate limiting algorithm, consensus protocol.
  3. Walk through **every tool and protocol**: API protocol, serialization format, transport security, deployment strategy, IaC tool, CI/CD pipeline, monitoring stack.
  4. For each one, verify a **Decision Record Table** exists (from earlier steps) that documents:
     - At least 2 alternatives that were considered
     - Pros and cons for **every** alternative (not just the chosen one)
     - The specific option chosen
     - A rationale explaining *why* it was chosen, referencing system requirements, calculations, or constraints
     - Trade-offs explicitly accepted
     - Why each rejected alternative was not suitable
  5. If any decision is **missing**, create its Decision Record Table now.
  6. Produce a **Consolidated ADR Index** — a numbered list (ADR-001, ADR-002, …) that cross-references each decision to the section of the markdown artifact where it appears.
- **Format:** Use the ADR template from [decision-log-evolution.md](file:///Users/luiz1361/gh_jae-labs/skills/skills/system-design-specialist/references/decision-log-evolution.md) §2 for the consolidated log.

### Step 17: Self-Review against Fitness Functions & Self-Critique
- **Goal:** Validate the architecture against the non-functional requirements.
- **Guideline:** Score each non-functional requirement (e.g., Latency, Availability, Scalability) against the current design. 
- **Self-Critique Questions (The Architecture Review):**
  - What is the most likely failure mode of this design?
  - What is the biggest/riskiest assumption we have made?
  - What is the highest cost component of this system?
  - What would be the hardest data/schema migration to execute?
  - What component would need to be completely redesigned first if traffic grows $10\times$?
  - What is the biggest unknown or third-party dependency risk?

### Step 18: Risks & Open Questions
- **Goal:** Acknowledge unknowns, high-risk assumptions, and future tasks.
- **Guideline:** Explicitly list unresolved questions, third-party dependencies, and testing challenges.

---

> [!IMPORTANT]
> **PHASE 3 GATE (CHALLENGE):**
> *Would this design survive a hostile environment (failure analysis, security threat modeling, cost limit, and operational complexity)?* **Ask the user to review the self-critique and align on final open tasks.**
