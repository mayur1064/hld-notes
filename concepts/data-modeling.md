# Data Modeling — Revision Notes
> Source: https://www.hellointerview.com/learn/system-design/core-concepts/data-modeling

---

## What is Data Modeling?

- Defining how application data is **structured, stored, and related**
- Deciding what **entities** exist, how they're **identified**, and how they **connect**
- In interviews: design something **clear, functional, and aligned with requirements** — not a production-perfect schema

### When it comes up in the Delivery Framework
1. **Requirements gathering** → identify core entities (map 1:1 to tables/collections)
2. **High-Level Design** → sketch a basic schema with key fields, relationships, and indexing/partitioning notes

---

## Database Model Options

> Default choice: **Relational (SQL) / PostgreSQL** unless requirements clearly signal otherwise.

### 1. Relational Databases (SQL)
- Data in **tables** with fixed schemas; rows = entities, columns = attributes
- Enforce relationships via **foreign keys**; provide **ACID** guarantees
- Best for most system design problems (social media, e-commerce, etc.)
- Handles complex queries with **JOINs** — but multi-table joins can become **performance traps** at scale
- Scalability concerns are often **exaggerated** — read replicas, sharding, connection pooling, caching help enormously
- **Examples:** PostgreSQL, MySQL, SQLite

### 2. Document Databases
- Stores data as **JSON-like documents** with flexible schemas
- Related data is **embedded** within documents (eliminates joins, but updates are harder)
- Use when: schema changes frequently, deeply nested data, records with vastly different structures
- In interviews: **rarely the right answer** since functional requirements are usually well-scoped
- Data modeling impact: aggressive **denormalization**, embedding over referencing
- **Examples:** MongoDB, Firestore, CouchDB

### 3. Key-Value Stores
- Simple **exact-key lookups** — extremely fast, limited querying
- Use for: **caching, session storage, feature flags**, high-write low-complexity scenarios
- Often used **alongside SQL** (not instead of): SQL as source of truth + Redis cache for hot data
- Data modeling impact: **very flat schema**, heavy denormalization, duplicate data per access pattern
- **Examples:** Redis, DynamoDB, Memcached

### 4. Wide-Column Databases
- Rows can have **different sets of columns**; optimized for **massive write-heavy** workloads
- Rows with the same partition key stored together → fast appends, efficient range scans
- Use for: **time-series data, telemetry, event logging, IoT sensor data**
- Data modeling impact: design around **query patterns**, duplicate data per access pattern, time is first-class
- **Examples:** Cassandra, HBase

### 5. Graph Databases
- Data as **nodes and edges**, optimized for relationship traversal
- Use for: social networks, recommendation engines (in theory)
- **Almost never the right answer in interviews** — even Facebook uses MySQL for their social graph
- Adds operational complexity without real benefit for most interview problems
- **Examples:** Neo4j, Amazon Neptune

---

## Schema Design Fundamentals

### Step 1: Start with Requirements

Three key factors drive schema design:

| Factor | What it affects |
|---|---|
| **Data volume** | Where data can physically live; may need multiple stores |
| **Access patterns** | Most important — drives indexing, denormalization, sharding decisions |
| **Consistency requirements** | Financial data → strong consistency (ACID); activity feeds → eventual consistency is fine |

> Tip: For each API endpoint ask "what queries will I need to support?"  
> Explicitly tie schema choices back to these factors in the interview.

---

### Step 2: Entities, Keys & Relationships

- Identify **core entities** → map each to a table/collection
- Assign **primary keys** using system-generated IDs (e.g., `user_id`, `post_id`), not business data like emails
- Use **foreign keys** to reference related entities

**Example schema (social media app):**
```
users:    id (PK), username, email, created_at
posts:    id (PK), user_id (FK → users.id), content, created_at
comments: id (PK), post_id (FK → posts.id), user_id (FK → users.id), content, created_at
likes:    user_id (FK → users.id), post_id (FK → posts.id)
```

**Relationship types:**
- **1:N (One-to-many):** user has many posts; post has many comments
- **N:M (Many-to-many):** users like many posts; posts liked by many users (junction table)
- **1:1 (One-to-one):** rare — often a sign the tables should be merged

**Foreign key trade-offs:**
- Enforce **referential integrity** (no orphaned records)
- Come at a **write cost** (DB validates every insert/update)
- At very large scale some companies drop FK enforcement and handle it at the application level

**Constraints:**
- Use `NOT NULL`, `UNIQUE`, `CHECK` to enforce correctness at DB level
- Protect data quality, but add write overhead

---

### Step 3: Indexing for Access Patterns

- Indexes let the DB find records **without a full table scan**
- Define indexes based on **your most important query patterns**

**Example indexes (social media):**
```
posts.user_id              → find all posts by a user
posts.created_at           → load posts chronologically
(user_id, created_at)      → efficiently load a user's recent posts (composite index)
```

> Tip: Connect every index to a specific API endpoint in the interview.  
> "GET /users/{id}/posts needs an index on posts.user_id."

---

### Step 4: Normalization vs. Denormalization

**Normalization** — each piece of data stored in **exactly one place**
- Prevents update anomalies (change username in one place, not everywhere)
- Start here by default

**Denormalization** — duplicate data for read performance
- Risk: updating denormalized data in multiple places can cause inconsistency
- Only denormalize when there's a clear performance need

**When denormalization makes sense:**
- Analytics/reporting (data changes infrequently)
- Event logs / audit trails (capturing a snapshot in time)
- Heavily read-optimized systems (search engines)

**Alternative:** Use a **cache** with a denormalized view in front of a normalized source of truth → best of both worlds.

---

### Step 5: Scaling and Sharding

- When data is too large for one machine → **shard** across multiple machines
- Key rule: **shard by the primary access pattern**

**Example:** If you mostly query "posts by user" → shard by `user_id`  
→ keeps a user's posts on the same DB, avoids cross-shard joins

**Anti-pattern — time-range sharding for write-heavy systems:**
- All current writes hit the same shard → **hot shard problem**
- Time-range partitioning works for read-heavy archival/analytics workloads instead

**Cross-shard queries:**
- Avoid whenever possible — expensive and complex
- Timeline queries (posts from multiple followed users) hit multiple shards → plan around this

> Shard key choice is often **permanent** — think carefully based on primary access patterns.

---

## Interview Checklist (Step-by-Step)

1. **Outline core entities** early during requirements gathering
2. When introducing a DB component in high-level design:
   - [ ] Choose the **type of database** (default: relational/PostgreSQL)
   - [ ] List **columns** needed per entity to satisfy functional requirements
   - [ ] Specify **primary keys** and **foreign keys** for each relationship
   - [ ] Identify which columns need **indexes**
   - [ ] Decide if **denormalization** is needed for performance
   - [ ] Consider **sharding** — if yes, choose a shard key matching the main access pattern

---

## Common Mistakes to Avoid

| Mistake | Better Approach |
|---|---|
| Choosing exotic DB types (graph, document) to seem sophisticated | Default to SQL unless requirements clearly dictate otherwise |
| Normalizing everything at the cost of read performance | Start normalized, denormalize selectively with a clear reason |
| Time-range sharding for write-heavy tables | Shard by entity ID (user_id, etc.) |
| Using business data (email) as primary key | Use system-generated IDs |
| Mentioning complex multi-table joins without addressing performance | Note potential performance issues and mitigations (indexes, caching, denormalized views) |
| Dropping foreign keys without justification | Mention the trade-off explicitly if you do |

---

## Quick Reference: DB Type Decision Tree

```
Is schema frequently changing or data deeply nested?
  └─ Yes → Document DB (MongoDB)
  
Mostly exact-key lookups, caching, or sessions?
  └─ Yes → Key-Value Store (Redis)
  
Massive write volume, time-series, or IoT data?
  └─ Yes → Wide-Column (Cassandra)
  
Social graph traversal?
  └─ Almost never needed → Use SQL anyway (Facebook does)
  
Everything else?
  └─ Relational SQL (PostgreSQL) ✓
```
