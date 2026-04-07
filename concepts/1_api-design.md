# API Design
> Source: https://www.hellointerview.com/learn/system-design/core-concepts/api-design

---

## Key Mindset for Interviews
- Most interviewers **don't care about perfect API design** — they want to see you design a reasonable API and **move on** to harder architectural problems.
- Spend **no more than 5 minutes** on API design in the interview.
- API design matters more for **frontend/product roles** and **junior roles**.

---

## API Types

### Decision Flowchart
```
Is the API client-facing or internal?
├── Internal → RPC (gRPC)
└── Client-facing →
        Is over/under-fetching a concern?
        ├── Yes → GraphQL
        └── No  → REST (default)
```

> **Default to REST** — it works for 90% of use cases.

---

### 1. REST (Representational State Transfer)
- Uses standard HTTP methods on **resource URLs**.
- Best for **CRUD operations** in web/mobile apps.
- **Default choice** unless there's a specific reason not to.

#### Resource Modeling
- Resources = **core entities** (nouns, plural): `events`, `venues`, `tickets`, `bookings`.
- Resources represent **things**, not actions.

```
GET  /events                    # list all events
GET  /events/{id}               # get specific event
GET  /venues/{id}               # get specific venue
GET  /events/{id}/tickets       # tickets for an event (nested — relationship required)
POST /events/{id}/bookings      # create booking for an event
GET  /bookings/{id}             # get specific booking
```

**Nesting vs. Flat:**
| Approach | When to use |
|---|---|
| Nested `/events/{id}/tickets` | Relationship is **required** — query doesn't make sense without it |
| Flat `/tickets?event_id=123` | Filter is **optional** — you might want all tickets without filter |

---

#### HTTP Methods

| Method | Purpose | Idempotent? | Notes |
|---|---|---|---|
| `GET` | Retrieve data (no side effects) | ✅ Yes | Safe |
| `POST` | Create new resource | ❌ No | Multiple calls = multiple resources |
| `PUT` | Replace entire resource (or create) | ✅ Yes | Send full object |
| `PATCH` | Partial update | ⚠️ Not guaranteed | Depends on implementation |
| `DELETE` | Remove resource | ✅ Yes | Repeated calls leave same state |

**Idempotency**: Repeated requests leave the server in the **same final state**.  
Critical when networks fail and clients **retry** — you don't want duplicate bookings.

---

#### Passing Data to APIs

| Location | Purpose | When to use |
|---|---|---|
| **Path parameter** `/events/123` | Identify a specific resource | Required to identify the resource |
| **Query parameter** `?city=NYC&page=2` | Filter, sort, paginate | Optional modifiers |
| **Request body** `{ ... }` | Complex/sensitive payload | Creating or updating data |

**Example:**
```
POST /events/123/bookings?notify=true
{
  "tickets": [
    {"section": "VIP", "quantity": 2}
  ],
  "payment_method": "credit_card"
}
```
- `123` → path param (required to identify the event)
- `notify=true` → query param (optional behavior)
- Body → actual booking payload

---

#### Response / Status Codes
Response = **status code** + **response body** (typically JSON).

| Code | Meaning |
|---|---|
| `200` | OK — success |
| `201` | Created — new resource created |
| `400` | Bad Request — client error |
| `401` | Unauthorized — authentication required |
| `404` | Not Found |
| `429` | Too Many Requests — rate limit exceeded |
| `500` | Internal Server Error |

> Key distinction: **4xx = client errors**, **5xx = server errors**. You don't need to memorize every code.

---

### 2. GraphQL
- Single endpoint; client specifies **exactly** what data it needs.
- Solves **over-fetching** (client gets too much data) and **under-fetching** (client needs multiple requests).
- Originated at Facebook (2012) for mobile vs. web data needs.

#### When to Use in Interviews
- Interviewer mentions **"mobile app needs different data than web app"**.
- Keywords: **"flexible data fetching"**, **"avoiding over/under-fetching"**.
- Frontend needs to iterate quickly without backend changes.

#### Example Query
```graphql
query {
  event(id: "123") {
    name
    date
    venue { name address }
    tickets { section price available }
  }
}
```

#### Schema Design
```graphql
type Event {
  id: ID!
  name: String!
  date: DateTime!
  venue: Venue!
  tickets: [Ticket!]!
}

type Query {
  event(id: ID!): Event
  events(limit: Int, after: String): [Event!]!
}
```

#### N+1 Problem (Key GraphQL Gotcha)
- Querying 100 events with venues → 1 query for events + 100 queries for venues = **101 DB queries**.
- Solution: **DataLoader / batching** patterns — adds complexity.

#### Tradeoffs
| ✅ Pros | ❌ Cons |
|---|---|
| Clients get exactly what they need | Complex to implement (query parsing, schema validation) |
| Reduces round trips | N+1 query problem |
| Frontend can iterate without backend changes | Harder caching strategies |

---

### 3. RPC (Remote Procedure Call)
- **Action-oriented** (call functions) vs. REST's resource-oriented approach.
- Most popular: **gRPC** (uses Protocol Buffers + HTTP/2).
- Much faster than JSON-over-HTTP for service-to-service communication.

#### How RPC Looks
```
// Instead of GET /events/123
getEvent(eventId: "123")

// Instead of POST /events/123/bookings
createBooking(eventId: "123", userId: "456", tickets: [...])
```

#### Protocol Buffers (protobuf)
- Define service in a `.proto` file → gRPC generates client/server code in multiple languages.
- Provides **compile-time type safety**.

```protobuf
service TicketService {
  rpc GetEvent(GetEventRequest) returns (Event);
  rpc CreateBooking(CreateBookingRequest) returns (Booking);
}

message Event {
  string id = 1;
  string name = 2;
  int64 date = 3;
}
```

#### When to Use RPC in Interviews
- **Internal service-to-service** communication (not user-facing APIs).
- Performance is critical (binary serialization >> JSON).
- Polyglot environments (services in different languages).
- Bidirectional **streaming** needed (gRPC supports it).
- Interviewer mentions **microservices** or **internal APIs**.

> In interviews, focus on **user-facing APIs** in the API step. Just mention internal services use RPC during high-level design.

---

## Common API Patterns

### Pagination
Essential when datasets are large — you can't return everything at once.

#### Offset-Based Pagination
```
GET /events?offset=20&limit=10    # returns records 21-30
```
- Simple, intuitive.
- ❌ Problem: if new records are added mid-pagination, you may see **duplicates or miss records**.
- Fine for most interview answers.

#### Cursor-Based Pagination
```
# First request
GET /events?limit=10
→ Response: { "events": [...], "next_cursor": "abc123xyz" }

# Next request
GET /events?cursor=abc123xyz&limit=10
```
- Cursor = encoded pointer to a specific record (ID or timestamp).
- ✅ Stable — not affected by new records being inserted.
- ❌ Can't easily "jump to page 5".
- Use when dealing with **real-time data** or **high-volume scenarios**.

> Interviewers care more that you **remembered to include pagination** than which type you chose.

---

### Versioning Strategies

#### URL Versioning (Recommended for interviews)
```
/v1/events
/v2/events
```
- Explicit, easy to understand and route.
- Easy to test in browsers.

#### Header Versioning
```
Accept-Version: v2
```
- Cleaner URLs, follows HTTP standards better.
- Harder to test/debug; less obvious to developers.

> Most interview solutions skip versioning — it's rarely a focus unless explicitly asked.

---

## Security Considerations

### Authentication vs. Authorization
- **Authentication**: Who are you? (verify identity)
- **Authorization**: What are you allowed to do? (check permissions)

### API Keys vs. JWT Tokens

| | API Keys | JWT Tokens |
|---|---|---|
| Best for | Server-to-server, 3rd party developer access | User-facing web/mobile apps |
| How it works | Long random string stored in DB | Signed token encoding user info |
| DB lookup on each request | ✅ Yes | ❌ No (self-contained) |
| Carries user context | ❌ No | ✅ Yes (user_id, role, expiry) |
| Stateless | ❌ | ✅ |

**JWT Payload example:**
```json
{
  "user_id": "123",
  "email": "john@example.com",
  "role": "customer",
  "exp": 1640995200
}
```

**Interview tip**: Call out which endpoints require authentication. Say you'd use JWT for user sessions or store sessions in a DB.

---

### Role-Based Access Control (RBAC)
Assign **roles** to users and **permissions** to roles.

```
customer       → book tickets, view own bookings
venue_manager  → create events, view sales for their venues
admin          → access everything
```

**Per-request check:**
1. Is the user **authenticated**? (valid JWT)
2. Is the user **authorized**? (owns this resource OR has admin role)

> In interviews: just mention which endpoints are accessible by which roles. Usually not a deep focus.

---

### Rate Limiting
Prevents abuse (malicious attacks and accidental overuse).

- **Per-user**: 1000 requests/hour per authenticated user
- **Per-IP**: 100 requests/hour for unauthenticated requests
- **Endpoint-specific**: 10 booking attempts/minute (anti-scalping)
- Implemented at **API gateway** level or app middleware.
- Return **`429 Too Many Requests`** when limit exceeded.

> In interviews: a simple mention ("we'll add rate limiting to prevent abuse") is sufficient.

---

