# Sharding — Revision Notes

> Source: https://www.hellointerview.com/learn/system-design/core-concepts/sharding

---

## 1. Partitioning vs. Sharding

| | Partitioning | Sharding |
|---|---|---|
| Data location | Single machine | Multiple machines |
| Purpose | Organise data for efficiency | Scale beyond one machine's limits |

**Types of partitioning:**
- **Horizontal** — Split rows across partitions (e.g., one partition per year). Same columns, fewer rows.
- **Vertical** — Split columns across partitions (e.g., frequently accessed vs. rarely accessed columns). Same rows, fewer columns.

> In practice, engineers use "partitioning" and "sharding" interchangeably. Be explicit about whether data lives on one machine or many.

---

## 2. What is Sharding?

- Horizontal partitioning **across multiple machines**.
- Each **shard** is an independent database with its own CPU, memory, storage, and connection pool.
- Together all shards form the complete dataset.
- Enables both **storage** and **read/write throughput** to scale horizontally.

When to shard: when a single database (even the largest cloud instance, e.g. Aurora ~256 TB) can no longer handle storage, write throughput, or read throughput.

---

## 3. Choosing a Shard Key

A shard key defines how data is grouped and distributed. Getting it wrong causes hotspots and expensive cross-shard queries.

### Properties of a good shard key

| Property | Why it matters |
|---|---|
| **High cardinality** | More unique values → more shards possible |
| **Even distribution** | Prevents one shard from being overloaded |
| **Aligns with Acess Pattern** | Common queries should hit a single shard |

### Examples

| Shard Key | Verdict | Reason |
|---|---|---|
| `user_id` | ✅ Good | High cardinality, even spread, most queries are user-scoped |
| `order_id` | ✅ Good | High cardinality, queries are order-scoped |
| `is_premium` (boolean) | ❌ Bad | Only 2 possible shards; highly uneven |
| `created_at` | ❌ Bad | All new writes go to the latest shard → write hotspot |

---

## 4. Sharding Strategies

### 4.1 Range-Based Sharding
- Assign contiguous value ranges to shards.
  ```
  Shard 1 → User IDs 1–1M
  Shard 2 → User IDs 1M–2M
  Shard 3 → User IDs 2M–3M
  ```
- **Pros:** Simple; efficient range scans hit one shard.
- **Cons:** Uneven load if access patterns cluster around ranges (e.g., time-based keys → all writes hit latest shard).
- **Best for:** Multi-tenant systems where each tenant queries its own range.

### 4.2 Hash-Based Sharding (Default)
- Apply a hash function to the shard key, then use modulo to pick a shard.
  ```
  shard = hash(user_id) % 4
  ```
- **Pros:** Even data distribution across shards.
- **Cons:** Adding/removing shards invalidates most assignments (`% 4` → `% 5` moves almost everything). Mitigated by **consistent hashing**.
- **Best for:** Most general use cases. Assume this unless you explicitly say otherwise.

### 4.3 Directory-Based Sharding
- A lookup table maps each key to a shard.
  ```
  User 15  → Shard 1
  User 87  → Shard 4
  User 204 → Shard 2
  ```
- **Pros:** Maximum flexibility; can move hot keys to dedicated shards.
- **Cons:** Every request needs a directory lookup (extra latency); directory is a **single point of failure**.
- **Best for:** Rarely the right answer in interviews — use only when flexibility is explicitly required.

---

## 5. Challenges of Sharding

### 5.1 Hot Spots & Load Imbalance
- **Celebrity problem:** `user_id` sharding puts Taylor Swift's traffic on one shard — 1000× more requests than a normal user.
- **Time-based hotspot:** All new writes go to the newest shard; old shards handle only reads.

**Mitigations:**
- **Isolate hot keys** to dedicated shards (via directory-based routing).
- **Compound shard keys:** `hash(user_id + date)` — spreads one user's data across shards over time.
- **Dynamic shard splitting:** MongoDB's balancer auto-splits/migrates chunks; Vitess supports operator-driven online resharding.

### 5.2 Cross-Shard Operations
- Queries that don't filter on the shard key must fan out to **all shards**, wait for all responses, and aggregate results.
- Example: "Top 10 posts globally" → query all 64 shards, merge results = 64× network calls.

**Mitigations:**
| Strategy | When to use |
|---|---|
| **Cache results** | Queries where eventual consistency is acceptable (leaderboards, trending) |
| **Denormalize** | Store related data together on one shard to avoid cross-shard joins |
| **Accept the hit** | Rare admin queries that run infrequently |

> In interviews, if you say "we'll query all shards and aggregate" for a *common* use case, pause and reconsider the design.

### 5.3 Maintaining Consistency
- Single-database ACID transactions don't work across shards.
- **Two-Phase Commit (2PC):** Coordinator prepares all shards then commits. Correct but slow and fragile (coordinator failure = system stuck).

**Better alternatives:**
| Approach | Description |
|---|---|
| **Design to avoid cross-shard txns** | Keep all of a user's data on their shard — best option |
| **Saga pattern** | Sequence of independent steps each with a compensating rollback action |
| **Eventual consistency** | Accept brief inconsistency for non-critical data (e.g., follower counts) |

> If you constantly need distributed transactions, you likely chose the wrong shard key or shard boundaries.

---

## 6. Sharding in Modern Databases

You rarely implement sharding from scratch:

| Database | Sharding Mechanism |
|---|---|
| **Cassandra** | Murmur3Partitioner with virtual nodes (consistent hashing over token ranges) |
| **DynamoDB** | Hashes partition key internally; auto-splits/merges partitions |
| **MongoDB** | Range-based chunks on shard key; background balancer rebalances |
| **Vitess** (MySQL) | Query routing + operator-driven online resharding |
| **Citus** (Postgres) | Sharding layer handling routing and cross-shard ops |
| **Aurora / Spanner** | Built-in distributed SQL with automatic sharding |

In interviews: *"We'll use DynamoDB with `user_id` as the partition key"* is sufficient unless asked to go deeper.

---

## 7. Sharding in System Design Interviews

### When to bring up sharding
Only after proving a single database won't scale. Trigger points:
- **Storage:** "500M users × 5KB = 2.5TB. Fine now, but at 10× we need to shard."
- **Write throughput:** "50K writes/sec exceeds what a single DB can handle."
- **Read throughput:** "100M DAU making multiple queries each — even with read replicas we need to distribute."

> **#1 mistake:** Introducing sharding before proving it's necessary.

### What to say (template)

1. **Propose a shard key** based on access patterns.
   > *"Most queries are user-centric, so I'd shard by `user_id`."*

2. **Choose a distribution strategy.**
   > *"Hash-based sharding with consistent hashing for even distribution."*

3. **Call out the trade-offs.**
   > *"Global queries like 'trending posts' require fan-out across all shards. We'd cache that result and pre-compute it with a background job."*

4. **Address growth.**
   > *"Start with 64 shards. Consistent hashing means adding shards only moves a fraction of the data."*

---
