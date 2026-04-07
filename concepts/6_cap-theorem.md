# CAP Theorem

> Source: https://www.hellointerview.com/learn/system-design/core-concepts/cap-theorem

---

## What is CAP Theorem?

In a distributed system, you can only guarantee **two out of three** properties:

| Property | Description |
|---|---|
| **Consistency (C)** | All nodes see the same data at the same time. Every read returns the most recent write. |
| **Availability (A)** | Every request to a non-failing node receives a response (but not necessarily the latest data). |
| **Partition Tolerance (P)** | The system continues to operate despite network failures/message loss between nodes. |

> **Key distinction:** CAP consistency ≠ ACID consistency. CAP consistency is about all nodes seeing the same data; ACID consistency is about valid state transitions.

---

## The Practical Reality

**Partition tolerance is non-negotiable** in any real distributed system — network failures will happen.

Therefore, CAP theorem in practice reduces to a single choice:

> **Consistency vs. Availability during a network partition**

---

## Choosing: Consistency vs. Availability

### Choose Consistency (CP) when…
Stale/inconsistent data would be **catastrophic**:
- **Ticket booking** — two users booking the same seat must be prevented
- **E-commerce inventory** — overselling a limited item must be prevented
- **Financial systems** — stock order books must reflect accurate, up-to-date prices

### Choose Availability (AP) when…
Eventual consistency is acceptable (most systems fall here):
- **Social media profiles** — seeing a stale profile picture for a few minutes is fine
- **Content platforms (Netflix)** — slightly outdated movie descriptions are acceptable
- **Review sites (Yelp)** — briefly showing old restaurant hours is better than an error

### The Key Question to Ask
> "Would it be catastrophic if users briefly saw inconsistent data?"
> - **Yes** → Choose Consistency
> - **No** → Choose Availability

---

## Impact on System Design

### If you prioritize Consistency:
- **Distributed Transactions** — Two-phase commit (2PC) to keep data stores in sync. Adds latency but guarantees consistency.
- **Single-Node DB** — Avoids replication issues by having one source of truth (limits scalability).
- **Technology choices:** PostgreSQL, MySQL, Google Spanner, DynamoDB (strong consistency mode)

### If you prioritize Availability:
- **Multiple Read Replicas** — Asynchronous replication; replicas may be slightly behind.
- **Change Data Capture (CDC)** — Propagates changes asynchronously to replicas, caches, and other systems.
- **Technology choices:** Cassandra, DynamoDB (multi-AZ configuration), Redis clusters

---

## In a System Design Interview

CAP theorem should be addressed **early** — during the non-functional requirements phase:

1. Align on functional requirements (features)
2. Define non-functional requirements — **start here with CAP**
3. Ask: *"Does this system need to prioritize consistency or availability?"*

This decision shapes your entire architecture.

---

## Advanced: Mixed Requirements (Senior/Staff Level)

Real-world systems often need **different consistency models for different features** within the same system.

### Example — Ticketmaster:
| Feature | Choice | Reason |
|---|---|---|
| Booking a seat | **Consistency** | Prevent double-booking |
| Viewing event details | **Availability** | Stale descriptions are acceptable |

### Example — Tinder:
| Feature | Choice | Reason |
|---|---|---|
| Matching | **Consistency** | Both users should see the match immediately |
| Viewing profiles | **Availability** | Briefly stale profile picture is acceptable |

---

## Consistency Spectrum

"Consistency" isn't binary — there's a spectrum from strongest to weakest:

| Model | Description | Example Use Case |
|---|---|---|
| **Strong Consistency** | All reads reflect the most recent write | Bank account balances |
| **Causal Consistency** | Related events appear in the same order to all users | Comments must appear after the post |
| **Read-Your-Own-Writes** | Users always see their own updates, others may see older versions | Social media profile updates |
| **Eventual Consistency** | System becomes consistent over time, may have temporary inconsistencies | DNS, most distributed DBs in AP mode |

---

## Quick Reference

```
CAP Theorem
├── P (Partition Tolerance) — Always required
└── Choose one during partition:
    ├── C (Consistency) → CP systems
    │   └── Return error instead of stale data
    └── A (Availability) → AP systems
        └── Return stale data instead of error
```

---

## TL;DR

> Ask yourself: **"Does every read need to return the most recent write?"**
> - **Yes** → Prioritize Consistency (CP)
> - **No** → Prioritize Availability (AP) with eventual consistency
