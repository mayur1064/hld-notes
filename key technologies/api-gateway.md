# API Gateway — Revision Notes

## What is an API Gateway?

- A **single entry point** for all client requests in a microservices architecture.
- Routes requests to appropriate backend services and handles cross-cutting concerns.
- Emerged as microservices replaced monoliths, creating the need for a centralized control point.

---

## Core Responsibilities — Request Lifecycle

### 1. Request Validation

- First step: check if requests are properly formatted.
- Validates: URL, required headers, request body format.
- Rejects malformed requests early (e.g., bad JSON, missing API key) — saves backend resources.

### 2. Middleware

- Configurable cross-cutting concerns applied before routing:
  - **Authentication** (e.g., JWT tokens)
  - **Rate limiting**
  - **SSL termination**
  - **IP allowlisting/denylisting**
- **Interview tip:** Most relevant are **auth, rate limiting, and IP filtering**. Don't spend too much time here — just say "routing and basic middleware" and move on.

### 3. Routing

- Gateway maintains a **routing table** mapping requests → backend services.
- Routing based on:
  - URL paths (e.g., `/users/*` → user-service)
  - HTTP methods
  - Query parameters
  - Request headers
- Example config:
  ```yaml
  routes:
    - path: /users/*
      service: user-service
      port: 8080
    - path: /orders/*
      service: order-service
      port: 8081
  ```

### 4. Response Transformation

- Most services use HTTP; some use **gRPC** internally.
- Gateway can **translate between protocols** (e.g., HTTP ↔ gRPC) so clients don't need to know.
- Transforms backend responses into the format the client expects.
- Example: internal gRPC response → JSON over HTTP to the client.


### 6. Caching

- Optionally cache responses before returning to client.
- Best for: frequently accessed, **non-user-specific** data with the same result for a given input.
- Strategies:
  - **Full Response Caching** — cache entire responses.
  - **Partial Caching** — cache parts that change infrequently.
  - **Cache Invalidation** — TTL-based or event-based.
- Storage: in-memory or distributed cache (e.g., Redis).

---

## Scaling an API Gateway

### Horizontal Scaling

- API Gateways are typically **stateless** → ideal for horizontal scaling.
- Add more instances behind a load balancer.
- Two levels of load balancing:
  - **Client → Gateway:** Handled by a dedicated LB (e.g., AWS ELB, NGINX).
  - **Gateway → Service:** The gateway itself can load-balance across service instances.
- **Interview tip:** A single "API Gateway + Load Balancer" box is usually sufficient. Don't get bogged down here.

### Global Distribution

- Deploy gateways closer to users (similar to CDN approach).
- Steps:
  1. **Regional deployments** in multiple geographic regions.
  2. **GeoDNS** to route users to the nearest gateway.
  3. **Configuration sync** to keep routing rules consistent across regions.

---

## Popular API Gateways

### Managed Services

| Service | Highlights |
|---|---|
| **AWS API Gateway** | REST & WebSocket, Lambda integration, CloudWatch, throttling |
| **Azure API Management** | OAuth/OIDC, policy-based config, developer portal |

---

## When to Use an API Gateway (Interview Guidance)

- **Use it** when you have a **microservices architecture** — almost essential to avoid tight client-service coupling.
- **Don't use it** for simple monolithic apps or single-client-type systems — adds unnecessary complexity.
- **Interview tip:** Understand the component, but **don't spend more than ~30 seconds** on it. Mention it handles routing and middleware, then move on to the more interesting parts of your design.
