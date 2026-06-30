# Sizing & Capacity Estimation Guide (NALSD & Back-of-the-Envelope)

A non-abstract system design requires grounding in physical limits. This guide details physical hardware capacities, latency figures, and standard calculation templates.

---

## 1. Latency Reference Table (Operational Physics)

Keep these latency numbers in mind when designing. They dictate where data must live to satisfy latency SLAs:

| Operation | Latency | Speed comparison |
|-----------|---------|------------------|
| L1 Cache Reference | 0.5 ns | - |
| L2 Cache Reference | 7 ns | 14x slower than L1 |
| Mutex Lock/Unlock | 25 ns | - |
| Main Memory Reference (RAM) | 100 ns | 200x slower than L1 |
| Send 1 KB over 1 Gbps network | 10 us | 100x slower than RAM |
| Read 4 KB randomly from SSD | 150 us | 1,500x slower than RAM |
| Read 1 MB sequentially from RAM | 250 us | 2.5x slower than SSD random |
| Round trip in same Data Center | 500 us (0.5 ms) | - |
| Read 1 MB sequentially from SSD | 1,000 us (1 ms) | 10,000x slower than RAM |
| HDD Seek | 10,000 us (10 ms) | 100,000x slower than RAM |
| Read 1 MB sequentially from HDD | 20,000 us (20 ms) | - |
| WAN Round Trip (NY to London) | 60,000 us (60 ms) | - |
| WAN Round Trip (CA to Singapore) | 150,000 us (150 ms) | - |

---

## 2. Hardware Resource Capacities (Rule-of-Thumb Limits)

When sizing your servers, databases, and caches, assume the following default capabilities per standard virtual machine instance:

### A. Compute (CPU)
- **Web Server:** A standard 4-8 core instance can handle **5,000 to 10,000 QPS** for stateless JSON routing/logic.
- **WebSocket Server:** A single server can hold **50,000 to 100,000 active concurrent connections** (limited by file descriptors and RAM overhead per connection, which is typically 10 KB to 50 KB).

### B. Memory (RAM / Caching)
- **Redis Cache:** A single standard cache instance handles **100,000 to 250,000 QPS** for GET/SET operations. 
- **Redis RAM capacity:** Usually capped at **64 GB to 128 GB** per instance (beyond this, garbage collection/compaction and replication lag become risks). Shard Redis for larger sets.

### C. Storage (Disk & IOPS)
- **SATA HDD:** ~100 random IOPS. Sequential throughput is ~100 MB/s.
- **SATA SSD:** ~50,000 random IOPS. Sequential throughput is ~500 MB/s.
- **NVMe SSD:** ~500,000+ random IOPS. Sequential throughput is ~3 GB/s.
- **Relational Databases (PostgreSQL/MySQL):** Handles **1,000 to 5,000 write QPS** and **10,000 read QPS** on a single node before needing replication/sharding.
- **NoSQL Databases (Cassandra/DynamoDB):** Capped at **10,000+ write QPS** per node, scaling linearly as nodes are added.

### D. Network Bandwidth
- **Server Network Interface:** Standard instances support **1 Gbps to 10 Gbps** NICs (125 MB/s to 1.25 GB/s throughput).
- **Public CDN Egress:** Scales almost infinitely, but bandwidth cost dominates.

---

## 3. Storage and Bandwidth Calculations

Use these formulas to calculate storage, network, and caching sizes:

### QPS & Throughput Formula
$$\text{Average QPS} = \frac{\text{DAU} \times \text{User Actions per Day}}{86,400 \text{ seconds}}$$
$$\text{Peak QPS} = \text{Average QPS} \times \text{Peak Factor} \quad (\text{Peak Factor} = 3 \text{ to } 5)$$

### Storage Volume Formula
$$\text{Storage/Day} = \text{Writes/Day} \times \text{Average Record Size} \times 1.3 \quad (30\% \text{ overhead for indexes/metadata})$$
$$\text{Storage/Year} = \text{Storage/Day} \times 365$$
$$\text{Total Storage} = \text{Storage/Year} \times \text{Retention Period (Years)} \times \text{Replication Factor}$$

### Caching Size Formula (The 80/20 Rule)
- 20% of the hot data (the working set) generates 80% of the read requests.
$$\text{Working Set Cache Size} = \text{Total Unique Records in Active Period} \times 0.20 \times \text{Record Size}$$

---

## 4. Cost Estimation Guidelines

A key aspect of NALSD is cost estimation. Use these rough cloud provider pricing assumptions:

- **Compute Instance (Standard VM):** ~$50/month per 2 vCPUs and 8 GB RAM.
- **Database Instance (Managed DB):** ~$150/month per 4 vCPUs and 16 GB RAM.
- **SSD Storage:** ~$0.10 per GB per month ($100/month per TB).
- **Network Egress (Transit to Public Internet):** ~$0.08 per GB transferred. **This is highly expensive at scale!**
- **S3 Blob Storage:** ~$0.023 per GB per month ($23/month per TB).

### Worked Cost Example: Image Upload Platform
- **Scale:** 10 million image uploads per day.
- **Average Image Size:** 2 MB.
- **Monthly Storage Growth:** $10\text{M} \times 2\text{ MB} \times 30\text{ days} = 600\text{ TB/month}$.
- **Storage Cost (S3):** $600\text{ TB} \times \$23/\text{TB} = \$13,800/\text{month}$ (increasing by $\$13.8\text{k}$ each month!).
- **Network Egress Cost:** If 100 million image views occur daily:
  - Daily egress bandwidth: $100\text{M} \times 2\text{ MB} = 200\text{ TB/day} = 6,000\text{ TB/month}$.
  - Egress Cost: $6,000,000\text{ GB} \times \$0.08/\text{GB} = \$480,000/\text{month}$!
  - **Mitigation:** Use a CDN! CDN caching reduces origin egress, dropping the price per GB by 4x to 10x.
