
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

REST (Representational State Transfer) is an architectural style for distributed systems.

It defines constraints that ensure:

- Scalability  
- Evolvability  
- Loose coupling  
- Clear separation of concerns  
- Operational stability  

REST focuses on system behavior under scale and change — not simply on HTTP or JSON.

---

## 1.2 REST Constraints

### 1.2.1 Client–Server Separation

Clients and servers evolve independently.

Backend services must not:

- Contain UI-specific logic  
- Depend on frontend implementation details  
- Break contracts when UI changes  

---

### 1.2.2 Statelessness

Each request must contain all required information.

The server MUST NOT store session state between requests.

Valid example:

```
Authorization: Bearer <JWT>
```

Forbidden:

- HTTP sessions  
- Sticky sessions  
- In-memory per-user state  

Statelessness enables horizontal scaling and resilience.

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

---

### 1.2.4 Uniform Interface

All services MUST follow consistent:

- URI naming conventions  
- Error model  
- Pagination format  
- Versioning strategy  

Uniformity reduces integration complexity and operational friction.

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

Never expose internal domain structures directly.

---

## 2.2 URI Design Rules

APIs MUST:

- Use plural nouns  
- Use lowercase  
- Use hyphen-separated names  
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

## 2.5 Orders API Example

### Create Order

```
POST /api/v1/orders
```

Headers:

```
Authorization: Bearer <JWT>
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

Request body:

```json
{
  "customerId": "CUST-123",
  "amount": 150.50,
  "currency": "EUR"
}
```

Response:

```
201 Created
Location: /api/v1/orders/9c4b6c7e-0d4e-4b5b-a7f0-92d9e5f1c222
```

Response body:

```json
{
  "id": "9c4b6c7e-0d4e-4b5b-a7f0-92d9e5f1c222",
  "customerId": "CUST-123",
  "amount": 150.50,
  "currency": "EUR",
  "status": "CREATED",
  "createdAt": "2026-02-19T10:00:00Z"
}
```

---

## 2.6 Standard Error Model

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

- Thread blocking  
- Pool exhaustion  
- Cascading failure  

---

## 3.2 Retry Policy

Retries are allowed only for:

- Network failures  
- HTTP 5xx responses  

Retries are forbidden for:

- 4xx responses  
- Validation errors  
- Non-idempotent POST operations  

Example:

```java
@Retry(name = "externalService")
```

---

## 3.3 Circuit Breaker

Circuit breakers prevent cascading failures.

Example:

```java
@CircuitBreaker(name = "externalService")
```

---

## 3.4 Rate Limiting

Services must return:

```
429 Too Many Requests
```

When limits are exceeded.

---

# 4. Performance Standards

## 4.1 Performance Targets

- Internal P95 latency < 200ms  
- External P95 latency < 400ms  
- Error rate < 1%  

---

## 4.2 Concurrency Handling

Tomcat configuration:

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

---

## 4.3 Async for Long Operations

Requests longer than 3 seconds MUST be asynchronous.

Pattern:

```
POST /reports  →  202 Accepted  
GET  /reports/{id}  →  status  
```

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

---

## 5.2 Authorization

Server-side enforcement is mandatory.

Example:

```java
@PreAuthorize("hasAuthority('SCOPE_orders:write')")
```

---

## 5.3 Input Validation

All DTOs must use Bean Validation.

---

## 5.4 Transport Security

- HTTPS mandatory  
- TLS 1.2 or higher  

---

# 6. Observability Requirements

## 6.1 Correlation ID

Header:

```
X-Correlation-ID
```

Must be generated if missing and propagated downstream.

---

## 6.2 Structured Logging

Logs must include:

- timestamp  
- level  
- service  
- traceId  
- spanId  
- correlationId  

Sensitive information (tokens, passwords, PII) must never be logged.

---

## 6.3 Metrics

Micrometer must expose:

- http.server.requests  
- latency percentiles  
- error rate  
- DB pool usage  
- thread pool usage  
- JVM memory metrics  

---

## 6.4 Distributed Tracing

Enable tracing:

```yaml
management.tracing.enabled=true
```

Trace context must propagate across services.

---

# Final Principle

This document defines the REST engineering baseline for the Foundation project.

All APIs must comply.  
Architectural deviations require review and approval.
