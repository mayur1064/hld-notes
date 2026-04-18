# Redis — Revision Notes

## What is Redis?

- **Data structure store** written in C
- **In-memory** and **single-threaded** → extremely fast and easy to reason about
- Key-value store where keys are strings, values can be any supported data structure
- Handles **~100k+ writes/sec**, read latency often in the **microsecond** range

## Core Data Structures

| Structure | Description |
|-----------|-------------|
| Strings | Basic key-value |
| Hashes | Objects / dictionaries |
| Lists | Ordered collections |
| Sets | Unordered unique elements |
| Sorted Sets | Priority queues / ordered by score |
| Bloom Filters | Probabilistic set membership (allows false positives) |
| Geospatial Indexes | Longitude/latitude based indexing |
| Time Series | Time-stamped data |
| Streams | Append-only logs (similar to Kafka topics) |

## Commands (Examples)

```
SET foo 1
GET foo          # Returns 1
INCR foo         # Returns 2 (atomic increment)
SADD myset "a"   # Add to set
ZADD leaderboard 500 "player1"  # Add to sorted set with score
GEOADD locations 13.36 52.52 "Berlin"
XADD mystream * name Sara       # Append to stream
```

## Infrastructure Configurations

| Mode | Description |
|------|-------------|
| **Single Node** | Simplest setup |
| **Replicated (HA)** | Primary + secondary for high availability |
| **Cluster** | Data sharded across multiple nodes via **hash slots** |

### Cluster Details

- Clients cache a map of **hash slots → nodes** (like a phone book)
- If a slot moves, server replies with `MOVED` and client refreshes via `CLUSTER SHARDS`
- Nodes maintain awareness of each other via **gossip protocol**
- **Key limitation**: all data for a given request must be on a **single node**
- How you structure your keys = how you scale your cluster

## Durability

- Redis does **not** guarantee durability by default (speed tradeoff)
- **AOF (Append-Only File)** can minimize data loss but isn't as strong as RDBMS guarantees
- **AWS MemoryDB** provides disk-based durability at slight speed cost
- If durability is critical, consider alternatives or MemoryDB

---

## Capabilities / Use Cases

### 1. Cache

- Most common use case
- Keys/values map directly to cache entries
- Use **TTL** on each key for automatic expiration and eviction
- Scales horizontally by adding cluster nodes
- Example: `SET product:123 '{"name":"Widget","price":9.99}'` with `EXPIRE`
- **Hot key problem**: uneven load can overwhelm a single node (see Shortcomings)

### 2. Distributed Lock

- Use atomic `INCR` + TTL for simple locks
  - `INCR lock_key` → if result is `1`, lock acquired; if `> 1`, someone else holds it
  - `DEL lock_key` to release
- For stronger guarantees: **Redlock algorithm** + **fencing tokens**
- Fencing tokens (monotonically increasing integers) prevent stale lock holders from corrupting data after GC pauses or network delays
- **Caveat**: if your primary DB provides consistency, prefer that over a distributed lock

### 3. Leaderboards

- **Sorted Sets** maintain ordered data, queryable in O(log N)
- `ZADD key score member` — add/update score
- `ZRANGE key 0 9` — get top 10
- `ZREMRANGEBYRANK key 0 -6` — trim to top 5
- Great for high-throughput, low-latency ranking

### 4. Rate Limiting

- **Fixed window**: `INCR key` per request, check against limit N, `EXPIRE` after window W
- **Sliding window**: store timestamps in a Sorted Set per key, remove old entries before counting; use Lua script for atomicity

### 5. Proximity / Geo Search

- `GEOADD key longitude latitude member`
- `GEOSEARCH key FROMLONLAT lon lat BYRADIUS radius unit`
- Uses **geohashes** under the hood → grid-aligned bounding box + second-pass radius filter
- Complexity: O(N + log M) — N = elements in radius, M = items in bounding shape

### 6. Event Sourcing / Work Queues

- **Streams**: append-only logs (like Kafka topics)
- `XADD` to add items
- **Consumer Groups** (`XREADGROUP`, `XCLAIM`) for distributed processing
- Failed worker? Another worker can `XCLAIM` the unprocessed message

### 7. Pub/Sub

- Real-time broadcast to multiple subscribers
- Commands: `SPUBLISH channel message`, `SSUBSCRIBE channel`
- **Sharded Pub/Sub** — scales across cluster nodes
- One connection per node (not per channel)
- **At-most-once delivery** — messages are NOT persisted; offline subscribers miss messages
- For durability/replay: use **Redis Streams**, Kafka, or pair with SNS/SQS
- **Don't roll your own Pub/Sub** — increases network hops, adds complexity (heartbeats, connection management)

---

## Shortcomings & Remediations

### Hot Key Problem

- One extremely popular key overloads a single node
- **Solutions**:
  1. **Client-side in-memory cache** — reduce Redis calls for repeated data
  2. **Key replication** — store same data under multiple keys, randomize requests to spread load
  3. **Read replicas** — dynamically scale with load

### General Limitations

- **Memory-intensive** — all data in RAM; costly for large datasets
- **Potential data loss** — without persistence configured, crashes lose data
- **Limited query capabilities** — no complex/relational queries
- **Single-node constraint for requests** — all data for a request must reside on one node

---

## Key Interview Tips

- Redis is **versatile** — learning it deeply covers many use cases (cache, locks, leaderboards, rate limiting, geo, pub/sub, streams)
- Recognize **hot key issues** and proactively design remediations
- Know when Redis is **not** the right choice (durability-critical, complex queries, large datasets that don't fit in memory)
- Understand **tradeoffs** of Pub/Sub vs Streams vs external brokers (Kafka, RabbitMQ)
- Fencing tokens + Redlock for safe distributed locking
- Key design = scaling strategy in Redis clusters
