# Bit.ly — URL Shortener (Revision Notes)

> Source: [Hello Interview — Bit.ly](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly)

---

## Functional Requirements

1. **Shorten URL** — Users submit a long URL and receive a shortened version
   - Optionally specify a **custom alias** (e.g. `short.ly/my-custom-alias`)
   - Optionally specify an **expiration date**
2. **Redirect** — Users access the original URL by visiting the shortened URL

**Out of scope:** User authentication, analytics (click counts, geo data)

---

## Non-Functional Requirements

1. **Uniqueness** — Each short code maps to exactly one long URL
2. **Low latency** — Redirection < 100ms
3. **High availability** — 99.99% uptime; availability >> consistency
4. **Scale** — 1B shortened URLs, 100M DAU

**Key insight:** Read-to-write ratio is ~1000:1 (heavily read-heavy). This drives caching strategy, DB choice, and service separation.

**Out of scope:** Real-time analytics consistency, spam/malicious URL filtering

---

## Core Entities

| Entity | Description |
|---|---|
| **Original URL** | The long URL the user wants to shorten |
| **Short URL** | The shortened URL returned to the user |
| **User** | The user who created the shortened URL |

### Data Model (Urls table)

| Column | Size |
|---|---|
| `short_code` (PK / custom alias) | ~8 bytes |
| `original_url` | ~100 bytes |
| `creationTime` | ~8 bytes |
| `expirationTime` (optional) | ~8 bytes |
| `createdBy` | variable |

~500 bytes/row → 1B rows ≈ **500 GB** (fits on a single modern SSD)

---

## API Design

### 1. Shorten a URL

```
POST /urls
{
  "long_url": "https://www.example.com/some/very/long/url",
  "custom_alias": "optional_custom_alias",       // optional
  "expiration_date": "optional_expiration_date"   // optional
}
→ { "short_url": "http://short.ly/abc123" }
```

### 2. Redirect to Original URL

```
GET /{short_code}
→ HTTP 302 Found (Redirect to original long URL)
```

**Why 302 over 301?**
- 302 (temporary) — browser does NOT cache; every request hits our server
- Allows updating/expiring links without stale browser caches
- Keeps door open for future analytics
- 301 (permanent) — browser caches redirect, bypasses our server on subsequent visits

---

## High-Level Design

![alt text](image-1.png)
---

## Deep Dives & Reasoning

### 1. Ensuring Short Code Uniqueness

#### Bad: Long URL Prefix
- Just taking a prefix of the URL — high collision rate, not viable

#### Great: Hash Function
- Hash the long URL (e.g. MD5/SHA-256), take first 7 chars, Base62-encode
- Pros: stateless, no coordination needed
- Cons: possible collisions (handle by retry with salt); short code length ~7 chars

#### Great: Unique Counter + Base62 Encoding (Chosen for Final Design)
- Maintain a global auto-incrementing counter
- Convert counter value to Base62 → short code (7 chars of Base62 = 62^7 ≈ **3.5 trillion** unique codes)
- Pros: guaranteed uniqueness, short codes, simple
- Cons: requires centralized counter coordination at scale

### 2. Fast Redirects (Scaling Reads)

#### Good: Database Index
- Add index on `short_code` (already PK) → O(log n) lookups instead of full table scan

#### Great: In-Memory Cache (Redis)
- Cache `short_code → original_url` in Redis
- Most URLs follow power-law distribution (few URLs get vast majority of clicks)
- Cache absorbs the hot read traffic; cache TTL ≤ URL expiration time for correctness

#### Great: CDN / Edge Computing
- Push redirect logic to CDN edge nodes for geo-distributed low-latency reads
- Further reduces load on origin servers

### 3. Scaling to 1B URLs / 100M DAU

#### Database Choice
- Write throughput is low (~100k new URLs/day ≈ ~1 write/sec)
- Read throughput offloaded to cache
- **Any reasonable DB works**: Postgres, MySQL, DynamoDB
- Single Postgres instance can handle 500 GB; shard later if needed

#### Service Separation (Read vs Write)
- Split into **Read Service** (handles redirects) and **Write Service** (handles URL creation)
- Scale each independently — far more Read Service instances needed
- API Gateway routes `GET /{short_code}` → Read Service, `POST /urls` → Write Service

#### Centralized Redis Counter for Writes
- Problem: horizontally scaled Write Services need a single source of truth for the counter
- Solution: centralized **Redis instance** stores the global counter
- Redis supports **atomic increment** (`INCR`) — single-threaded, sub-ms latency

#### Counter Batching (Optimization)
- Each Write Service instance requests a **batch** (e.g. 1000 values) from Redis at once
- Uses values locally until exhausted, then requests a new batch
- Reduces Redis network calls dramatically

#### High Availability
- **Redis**: Redis Sentinel or Redis Cluster with automatic failover; losing a few counter values is acceptable (uniqueness not continuity required); DB `UNIQUE` constraint on `short_code` is the ultimate safety net
- **Database**: Replication (primary-replica) for failover; periodic backups
- **Multi-region**: Allocate disjoint counter ranges per region (e.g. region A: 0–1B, region B: 1B–2B) to avoid cross-region coordination

#### Custom Alias Collision Prevention
- Prefix auto-generated codes with a character custom aliases can't use, OR use separate namespaces

#### Expired URL Cleanup
- Background job periodically deletes expired rows
- Cache TTL ≤ URL expiration time so stale entries are auto-evicted

---

