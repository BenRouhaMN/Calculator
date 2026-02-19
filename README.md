# Foundation Project
# REST API Engineering Standard

**Technology Stack**  
Java 21 · Spring Boot 3.x · Spring Security · Resilience4j · Micrometer · OpenTelemetry · TypeScript

---

# Table of Contents

1. REST Foundations  
2. API Contract & Design Standards  
3. Resilience Standards  
4. Performance Standards  
5. Security Standards  
6. Observability Requirements  

---

# 1. REST Foundations

## 1.1 What REST Is

REST (Representational State Transfer) is an architectural style for distributed systems defined by Roy Fielding.

REST is not:
- Simply using HTTP
- Simply returning JSON
- Simply exposing controllers

REST is a set of architectural constraints that ensure:

- Horizontal scalability
- Loose coupling
- Evolvability over time
- Clear separation of concerns
- Predictable behavior under failure

REST optimizes long-term system stability over short-term implementation speed.

---

## 1.2 REST Constraints

### 1.2.1 Client–Server Separation

The client and server must evolve independently.

Backend services MUST:
- Not embed UI logic
- Not depend on specific frontend behavior
- Not break contracts due to UI changes

---

### 1.2.2 Statelessness

Each request must contain all necessary information.

The server MUST NOT store session state between requests.

Valid example:

```
Authorization: Bearer <JWT>
```

Forbidden:
- HTTP sessions
- Sticky sessions
- In-memory per-user state

Statelessness enables horizontal scaling and simplifies failure recovery.

---

### 1.2.3 Cacheability

GET responses SHOULD define cache behavior.

Example:

```
Cache-Control: max-age=60
```

Mutable endpoints MUST disable caching:

```
Cache-Control: no-store
```

Improper caching configuration may cause stale reads or data leaks.

---

### 1.2.4 Uniform Interface

All Foundation APIs MUST follow consistent:

- URI structure
- Error format
- Pagination rules
- Versioning strategy
- Authentication model

Uniformity reduces integration cost and operational complexity.

---

### 1.2.5 Layered System

Clients must not be aware of infrastructure layers.

APIs must not expose:
- Internal hostnames
- Network topology
- Internal service identifiers

---

# 2. API Contract & Design Standards

## 2.1 Resource Modeling

A resource represents a stable business concept.

It is NOT:
- A database table
- A service method
- An internal entity

Example:

Internal domain:
- OrderEntity
- PricingEngine
- RiskEvaluation

External API:
- Order
- Payment

Internal domain models MUST NOT be exposed directly.

---

## 2.2 URI Design Rules

APIs MUST:

- Use plural nouns
- Use lowercase
- Use hyphen-separated words
- Include version prefix
- Avoid verbs
- Avoid nesting deeper than 3 levels

Correct examples:

```
GET /api/v1/orders
GET /api/v1/orders/{orderId}
```

Incorrect examples:

```
GET /getOrders
POST /createOrder
POST /processPayment
```

---

## 2.3 Identifier Strategy

Identifiers MUST be:

- Opaque
- Immutable
- Globally unique

Use UUID:

```java
@Id
@GeneratedValue
private UUID id;
```

Never expose auto-increment database IDs.

---

## 2.4 HTTP Method Semantics

### GET
- Must not modify state
- Must be idempotent
- Returns 200 or 404

### POST
- Creates resource
- Returns 201
- Must include Location header

### PUT
- Full replacement
- Idempotent

### PATCH
- Partial update
- Idempotent

### DELETE
- Idempotent
- Returns 204 or 200

---

## 2.5 Idempotency for Critical Operations

Required for:
- Payments
- Financial operations
- Order submission

Client must send:

```
Idempotency-Key: <UUID>
```

Example table:

```sql
CREATE TABLE idempotency_keys (
  key VARCHAR(100) PRIMARY KEY,
  request_hash VARCHAR(255),
  response_payload TEXT,
  created_at TIMESTAMP
);
```

The service MUST:
- Store the key
- Store the response
- Return stored response if duplicate request detected

---

## 2.6 Standard Error Model

All APIs MUST return a consistent error structure:

```json
{
  "timestamp": "2026-02-19T10:01:00Z",
  "correlationId": "abc-123",
  "code": "VALIDATION_ERROR",
  "message": "Amount must be positive",
  "details": [
    {
      "field": "amount",
      "issue": "must be greater than 0"
    }
  ]
}
```

Stack traces MUST NOT be exposed.

---

# 3. Resilience Standards

## 3.1 Timeouts

All outbound calls MUST define timeouts.

HTTP example:

```java
HttpClient.create()
    .responseTimeout(Duration.ofSeconds(3));
```

Database example:

```yaml
spring.datasource.hikari.connection-timeout=3000
```

Without timeouts:
- Threads block indefinitely
- Thread pool saturation occurs
- Cascading failure becomes likely

Timeout values must be shorter than gateway timeouts.

---

## 3.2 Retry Policy

Retries are allowed only for:
- Network failures
- HTTP 5xx responses

Retries are forbidden for:
- 4xx responses
- Validation errors
- Non-idempotent operations

Example:

```java
@Retry(name = "externalService")
```

Retries MUST use exponential backoff to prevent retry storms.

---

## 3.3 Circuit Breaker

Circuit breakers prevent repeated calls to failing dependencies.

Example:

```java
@CircuitBreaker(name = "externalService")
```

Without circuit breaker:
- Downstream failures propagate
- Threads remain blocked
- Service capacity collapses

---

## 3.4 Bulkhead Isolation

Separate thread pools must be used for:

- External HTTP calls
- Asynchronous background jobs

This prevents one dependency from consuming all available threads.

---

## 3.5 Rate Limiting

Services MUST return:

```
429 Too Many Requests
```

When request rate exceeds defined limits.

Rate limiting protects against:
- Traffic spikes
- Abuse
- Accidental overload

---

# 4. Performance Standards

## 4.1 Performance Targets

- Internal P95 latency < 200ms
- External P95 latency < 400ms
- Error rate < 1%

These targets must be monitored continuously.

---

## 4.2 Concurrency Handling

Tomcat configuration example:

```yaml
server.tomcat.threads.max=200
```

Database pool configuration:

```yaml
spring.datasource.hikari.maximum-pool-size=20
```

Rule:

```
Tomcat threads ≤ 4 × DB pool
```

Misalignment leads to thread starvation and latency amplification.

---

## 4.3 Avoid Long Running Synchronous Requests

Requests longer than 3 seconds MUST be asynchronous.

Pattern:

```
POST /reports  →  202 Accepted
GET  /reports/{id}  →  status
```

Long blocking operations consume valuable threads and reduce throughput.

---

## 4.4 N+1 Query Prevention

Avoid patterns like:

```java
orders.forEach(o -> o.getItems());
```

Use fetch joins:

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
```

---

## 4.5 Caching

Use distributed cache (Redis) or local cache (Caffeine).

Example:

```java
@Cacheable("orders")
```

Cache TTL must be explicitly defined.

---

# 5. Security Standards

## 5.1 Authentication

OAuth2 + JWT required.

Configuration example:

```yaml
spring.security.oauth2.resourceserver.jwt.issuer-uri=...
```

JWT must:
- Be signed
- Have expiration
- Validate issuer
- Validate audience

Expired or invalid tokens MUST be rejected.

---

## 5.2 Authorization

Server-side enforcement is mandatory.

Example:

```java
@PreAuthorize("hasAuthority('SCOPE_orders:write')")
```

Object-level authorization must also validate ownership when applicable.

---

## 5.3 Input Validation

All DTOs MUST use Bean Validation.

Invalid requests MUST be rejected before business logic execution.

---

## 5.4 Transport Security

- HTTPS mandatory
- TLS 1.2 or higher
- HSTS recommended

Plain HTTP must not be enabled in production.

---

## 5.5 Secrets Management

Secrets MUST NOT be stored in source code.

Use:
- Environment variables
- Secret managers
- Vault systems

---

## 5.6 Audit Logging

Sensitive operations MUST be auditable.

Example schema:

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY,
  user_id VARCHAR(100),
  action VARCHAR(100),
  resource_id VARCHAR(100),
  timestamp TIMESTAMP
);
```

Audit logs must be immutable.

---

# 6. Observability Requirements

## 6.1 Correlation ID

Header:

```
X-Correlation-ID
```

If missing, it MUST be generated.

It MUST be propagated to downstream services.

---

## 6.2 Structured Logging

Logs MUST include:

- timestamp
- level
- service
- traceId
- spanId
- correlationId

Sensitive information must never be logged.

---

## 6.3 Metrics

Micrometer MUST expose:

- http.server.requests
- latency percentiles (p50, p95, p99)
- error rate
- DB pool usage
- thread pool usage
- JVM memory metrics

Metrics must be exported to monitoring systems.

---

## 6.4 Distributed Tracing

Enable tracing:

```yaml
management.tracing.enabled=true
```

Trace context must propagate across service boundaries.

---

# Final Principle

This document defines the REST engineering baseline for the Foundation project.

All APIs MUST comply.

Architectural deviations require formal review and approval.
