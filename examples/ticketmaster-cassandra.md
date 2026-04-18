# Ticketmaster – Cassandra Data Modeling Example

> Source: [Hello Interview – Cassandra Deep Dive](https://www.hellointerview.com/learn/system-design/deep-dives/cassandra#example-ticketmaster)

---

## Problem Context

- **Use case**: Ticket browsing UI – shows an event venue's available seats, allows seat selection and checkout.
- **Consistency**: Strict consistency is **not** needed for browsing. Seat availability changes in real-time; actual availability is verified against a consistent DB only at checkout.
- **Availability**: Always showing the browsing UI is critical – most users browse, few actually buy.
- Cassandra fits well here: **AP-leaning** (availability + partition tolerance), high read throughput, eventual consistency is acceptable.

---

## Iteration 1 – Naive Schema

Partition by `event_id`, one row per seat:

```sql
CREATE TABLE tickets (
  event_id  bigint,
  seat_id   bigint,
  price     bigint,
  PRIMARY KEY (event_id, seat_id)
);
```

### Pros
- Single partition per event → simple, one-partition query.

### Problems
- **Large partitions** for events with 10,000+ seats → performance degradation.
- DB must aggregate (price totals, ticket counts) on every query – expensive for popular events with frequent reads.

---

## Iteration 2 – Add `section_id` to Partition Key

Model mirrors the actual Ticketmaster UX:
1. User first sees a **venue map with sections** (high-level availability/price per section).
2. User clicks a section → sees **individual seats**.

```sql
CREATE TABLE tickets (
  event_id    bigint,
  section_id  bigint,
  seat_id     bigint,
  price       bigint,
  PRIMARY KEY ((event_id, section_id), seat_id)
);
```

### Why This is Better
| Concern | Improvement |
|---|---|
| **Partition size** | Each partition holds only one section's seats (much smaller). |
| **Distribution** | Event data spreads across multiple nodes (one partition per section). |
| **Access pattern alignment** | Matches the UI – querying seats for a single section hits one partition. |

---

## Denormalized Summary Table – `event_sections`

With `tickets` partitioned by section, we lose the ability to get event-wide stats from a single query. Solution: **denormalize** into a summary table.

```sql
CREATE TABLE event_sections (
  event_id     bigint,
  section_id   bigint,
  num_tickets  bigint,
  price_floor  bigint,
  PRIMARY KEY (event_id, section_id)
);
```

- Partitioned by `event_id` → one partition serves the entire venue map view.
- Events have a low number of sections (usually < 100), so partition stays small.
- Stats (ticket count, price floor) don't need to be exact – eventual consistency is fine (Ticketmaster shows approximate totals like "100+").

---

## Key Takeaways

1. **Query-driven modeling** – Design the schema around application access patterns, not entity relationships. Cassandra has no JOINs.
2. **Partition key design** – Controls data distribution and which queries can be served from a single partition. Choose keys that match your most common query.
3. **Partition size awareness** – Avoid unbounded or very large partitions. Introduce composite partition keys (e.g., adding `section_id`) to bound partition size.
4. **Data denormalization** – Duplicate/summarize data across tables to serve different access patterns efficiently. Trade storage for read performance.
5. **Eventual consistency is OK for reads** – Use a strongly consistent store only at the point of actual transaction (checkout), not for browsing.

---

## Quick Recall Diagram

```
┌────────────────────────────────────────────────┐
│                  Browsing UI                   │
│                                                │
│  Venue Map (sections overview)                 │
│    └─ Query: event_sections WHERE event_id=X   │
│       (single partition, all sections)         │
│                                                │
│  Section Detail (individual seats)             │
│    └─ Query: tickets WHERE event_id=X          │
│              AND section_id=Y                  │
│       (single partition, one section's seats)  │
│                                                │
│  Checkout → hits consistent DB (not Cassandra) │
└────────────────────────────────────────────────┘
```
