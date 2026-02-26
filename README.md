Below is the **fully rewritten and deepened sections starting from 5**, structured, explicit, pedagogical, and enterprise-grade.

You can paste this directly after section 4 in your standard.

All sections are numbered and expanded for clarity and non-expert developers.

---

# 5. Transaction & Consistency Rules

This section defines how transactional integrity and system consistency must be guaranteed.

It applies to:

* REST APIs
* Service layer
* Async consumers
* Scheduled jobs
* Distributed flows

The objective is to prevent:

* Partial commits
* Duplicate side effects
* Inconsistent state
* Hidden data corruption
* Retry-induced duplication

---

## 5.1 Core Transaction Model

### 5.1.1 Spring Transaction Behavior

In Spring:

* RuntimeException → rollback
* CheckedException → no rollback

Because all platform exceptions extend RuntimeException, rollback behavior is deterministic.

---

## 5.2 Rollback Policy

### 5.2.1 Default Rollback Matrix

| Exception Type          | Rollback | Rationale              |
| ----------------------- | -------- | ---------------------- |
| BusinessException       | No       | No system failure      |
| TechnicalException      | Yes      | Infrastructure failure |
| SystemCriticalException | Yes      | Integrity risk         |

---

### 5.2.2 Forcing Rollback on BusinessException

If business logic modifies state before throwing:

```java
@Transactional(rollbackFor = BusinessException.class)
```

This must be justified in code comments.

---

## 5.3 Idempotency (Mandatory for Retryable Flows)

### 5.3.1 Why Idempotency Is Critical

Without idempotency:

* Retry may double-charge
* Retry may duplicate orders
* Retry may over-reserve stock

Retry without idempotency is forbidden.

---

### 5.3.2 When Idempotency Is Mandatory

* Retry enabled
* External dependency involved
* Payment flows
* Distributed transactions
* Event-driven reprocessing

---

### 5.3.3 Idempotency Implementation Pattern

#### Step 1 — Require Idempotency Key

Header:

```
X-Idempotency-Key
```

---

#### Step 2 — Persistence Table

```java
@Entity
@Table(
    name = "idempotency_record",
    uniqueConstraints = @UniqueConstraint(columnNames = "key")
)
public class IdempotencyRecord {

    @Id @GeneratedValue
    private Long id;

    @Column(nullable = false)
    private String key;

    @Column(nullable = false)
    private String responseHash;

    @Column(nullable = false)
    private Instant createdAt;
}
```

---

#### Step 3 — Service Guard

```java
@Transactional
public PaymentResponse processPayment(
        String key,
        PaymentRequest request) {

    Optional<IdempotencyRecord> existing =
            repository.findByKey(key);

    if (existing.isPresent()) {
        return reconstructResponse(existing.get());
    }

    PaymentResponse response = executePayment(request);

    repository.save(new IdempotencyRecord(
            key,
            hash(response),
            Instant.now()
    ));

    return response;
}
```

Guarantee:

Multiple identical requests → same logical result.

---

## 5.4 Partial Commit Prevention

### 5.4.1 Dangerous Pattern

```java
@Transactional
public void process() {
    saveOrder();
    callExternalService();
}
```

If external call fails:

* DB committed
* External state inconsistent

---

### 5.4.2 Approved Patterns

#### A. Outbox Pattern (Preferred)

1. Save business entity
2. Save outbox event
3. Commit transaction
4. Async publisher sends event

```java
@Entity
public class OutboxEvent {

    @Id @GeneratedValue
    private Long id;

    private String aggregateId;
    private String payload;
    private boolean processed;
}
```

Guarantee:

Atomic DB state + event persistence.

---

#### B. Saga Pattern (Distributed Flow)

Each step must define:

* Forward action
* Compensating action

Example:

1. Reserve inventory
2. Charge payment
3. Confirm order

If step 2 fails:

→ Release inventory

---

## 5.5 Consistency Escalation

If post-condition validation fails:

```java
if (!validateFinalState()) {
    throw new SystemCriticalException(
        "POST_CONDITION_FAILURE",
        "System state inconsistent",
        null
    );
}
```

SystemCriticalException must:

* Trigger rollback
* Emit alert
* Stop further processing

---

## 5.6 Retry & Transaction Interaction

### Forbidden

```java
@Transactional
@Retry(name = "externalService")
public void process() {
    saveToDb();
    callExternal();
}
```

Retry may re-execute after commit → duplication risk.

---

### Correct Design

Option 1: Move external call outside transaction
Option 2: Use Outbox pattern

---

## 5.7 Transaction Decision Matrix

| Scenario                    | Required Pattern        |
| --------------------------- | ----------------------- |
| Simple DB write             | @Transactional          |
| DB + external call          | Outbox                  |
| Multi-service orchestration | Saga                    |
| Retryable remote call       | Idempotency + Retry     |
| Integrity violation         | SystemCriticalException |

---

# 6. Retry & Resilience Policy

Retry is a resilience mechanism — not a recovery shortcut.

---

## 6.1 Retry Decision Matrix

| Exception                            | Retry |
| ------------------------------------ | ----- |
| BusinessException                    | No    |
| TechnicalException (retryable=false) | No    |
| TechnicalException (retryable=true)  | Yes   |
| SystemCriticalException              | No    |

---

## 6.2 Resilience4j Configuration

```yaml
resilience4j:
  retry:
    instances:
      externalService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2

  circuitbreaker:
    instances:
      externalService:
        slidingWindowSize: 20
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
```

---

## 6.3 Correct Usage

```java
@Retry(name = "externalService")
@CircuitBreaker(name = "externalService")
public Response callExternal() {
    ...
}
```

---

## 6.4 Forbidden Patterns

* Retry inside DB transaction
* Manual while(true) retry loops
* Retrying BusinessException
* Retrying non-idempotent payment calls

---

# 7. Observability Standard

Observability has three pillars:

* Logging
* Tracing
* Metrics

---

## 7.1 Logging (Structured JSON)

### Logback Configuration

```xml
<configuration>
    <appender name="JSON"
        class="net.logstash.logback.appender.LogstashConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON"/>
    </root>
</configuration>
```

---

## 7.2 Logging Rules

| Exception Type          | Log Level |
| ----------------------- | --------- |
| BusinessException       | WARN      |
| TechnicalException      | ERROR     |
| SystemCriticalException | ERROR     |

Full stacktrace logged internally only.

---

## 7.3 Correlation & Tracing

```java
@Component
public class CorrelationFilter extends OncePerRequestFilter {

    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain chain)
            throws ServletException, IOException {

        String correlationId =
                Optional.ofNullable(
                    request.getHeader("X-Correlation-ID"))
                .orElse(UUID.randomUUID().toString());

        MDC.put("correlationId", correlationId);
        response.setHeader("X-Correlation-ID", correlationId);

        chain.doFilter(request, response);
        MDC.clear();
    }
}
```

All logs must contain correlationId.

---

## 7.4 Metrics

```java
Counter.builder("exceptions_total")
       .tag("type", ex.getClass().getSimpleName())
       .tag("retryable", String.valueOf(ex.isRetryable()))
       .register(meterRegistry)
       .increment();
```

Alerts:

* Error rate > 5% over 5 minutes
* Any SystemCriticalException

---

# 8. Security & Information Exposure Rules

---

## 8.1 External API Restrictions

Responses MUST NOT expose:

* Stacktrace
* SQL queries
* Internal class names
* Hostnames
* Infrastructure topology
* Sensitive identifiers

---

## 8.2 Sanitized Error Messages

External error messages must:

* Be client-safe
* Avoid internal system references
* Avoid database details

---

## 8.3 Sensitive Data Protection

Logs MUST NOT contain:

* Passwords
* Access tokens
* Payment card numbers
* PII

Mask sensitive values before logging.

---

## 8.4 Validation Enforcement

* Use @Valid for request validation
* Reject malformed input early
* Never rely on exception for input validation

---

# 9. Async / Batch Handling

| Exception               | Behavior                      |
| ----------------------- | ----------------------------- |
| BusinessException       | Log WARN, acknowledge message |
| TechnicalException      | Retry with backoff            |
| SystemCriticalException | Stop consumer, trigger alert  |

Dead-letter queues required for exhausted retries.

---

# 10. Governance & Compliance

PR rejected if:

* Silent catch exists
* Controller handles error manually
* Retry without idempotency
* Partial commit risk
* No correlation ID
* No structured logging
* Stacktrace exposed in API
* No metrics emission

Quarterly audit required.

---

This is now fully expanded from section 5 onward, structured and clear for non-experts.

If you want, I can now:

* Expand sections 1–4 to same depth
* Provide UML diagrams
* Provide architecture flow diagrams
* Convert into version 1.1 enterprise final document


Below is the **fully rewritten and deepened sections starting from 5**, structured, explicit, pedagogical, and enterprise-grade.

You can paste this directly after section 4 in your standard.

All sections are numbered and expanded for clarity and non-expert developers.

---

# 5. Transaction & Consistency Rules

This section defines how transactional integrity and system consistency must be guaranteed.

It applies to:

* REST APIs
* Service layer
* Async consumers
* Scheduled jobs
* Distributed flows

The objective is to prevent:

* Partial commits
* Duplicate side effects
* Inconsistent state
* Hidden data corruption
* Retry-induced duplicate


Parfait.
Tu as raison. La section 8 (Security & Information Exposure) doit être **beaucoup plus rigoureuse**, surtout si ton standard est destiné à un environnement bancaire / régulé / production critique.

Je te fournis une **section 8 entièrement réécrite**, structurée, normative, détaillée, avec :

* principes de séparation des couches
* contrôle de l’exposition des erreurs
* règles de sanitization
* protection des données sensibles
* logging sécurisé
* anti-patterns
* mapping avec OWASP
* exigences auditables

Tu peux remplacer entièrement ta section 8 par celle-ci.

---

# 8. Security & Information Exposure Rules

This section defines mandatory security constraints related to exception handling.

Improper exception exposure is a major attack vector and violates:

* OWASP A05: Security Misconfiguration
* OWASP A01: Broken Access Control
* OWASP A09: Security Logging & Monitoring Failures

Failure to comply may expose:

* Internal architecture
* Database structure
* Infrastructure topology
* Sensitive business logic
* Personal data (PII)

---

# 8.1 Separation of Concerns: Domain vs Transport

Exception objects belong to the application layer.

HTTP response formatting belongs to the presentation layer.

Therefore:

* Exceptions MUST NOT embed HTTP concerns.
* Exceptions MUST NOT contain user-facing formatted messages.
* HTTP layer MUST sanitize outgoing messages.

---

# 8.2 External Error Response Policy

API responses MUST NEVER expose:

* Stacktrace
* Full exception message
* SQL statements
* Table names
* Class names
* Internal package names
* Hostnames
* IP addresses
* Dependency URLs
* Secrets or tokens
* Infrastructure identifiers
* Configuration paths

---

## 8.2.1 Forbidden Example

```json
{
  "errorCode": "DB_FAILURE",
  "message": "org.postgresql.util.PSQLException: ERROR: relation users does not exist"
}
```

This leaks:

* Database vendor
* Table name
* SQL structure

---

## 8.2.2 Required Format

```json
{
  "timestamp": "2026-02-26T10:15:30Z",
  "errorCode": "DB_FAILURE",
  "message": "Service temporarily unavailable",
  "retryable": true,
  "correlationId": "abc-123"
}
```

---

# 8.3 Sanitization Rules

## 8.3.1 Internal vs External Message

Each exception MAY have:

* Internal message (detailed)
* External message (sanitized)

Example design:

```java id="sani01"
public abstract class PlatformException extends RuntimeException {

    private final String errorCode;
    private final boolean retryable;
    private final String externalMessage;

    protected PlatformException(
            String errorCode,
            String internalMessage,
            String externalMessage,
            boolean retryable,
            Throwable cause) {

        super(internalMessage, cause);
        this.errorCode = errorCode;
        this.externalMessage = externalMessage;
        this.retryable = retryable;
    }

    public String getExternalMessage() {
        return externalMessage;
    }
}
```

Handler MUST return externalMessage only.

---

# 8.4 Sensitive Data Protection in Logs

Logs are often exported to:

* ELK
* Splunk
* Cloud monitoring systems

Logs MUST NOT contain:

* Passwords
* Authorization headers
* JWT tokens
* API keys
* Credit card numbers
* CVV
* Personal addresses
* National identifiers

---

## 8.4.1 Masking Strategy

Example utility:

```java id="mask01"
public class LogSanitizer {

    public static String mask(String value) {
        if (value == null) return null;

        return value.replaceAll(
            "(?i)(password|token|authorization)=([^&\\s]+)",
            "$1=****"
        );
    }
}
```

All request logging MUST pass through sanitizer.

---

# 8.5 Stacktrace Exposure Policy

Stacktraces:

* MUST be logged internally
* MUST NEVER be returned in API response
* MUST NOT appear in production HTML error pages

Spring configuration:

```yaml
server:
  error:
    include-stacktrace: never
```

---

# 8.6 Validation Hardening

All incoming payloads MUST be validated before business logic execution.

Use:

```java
public Response create(@Valid @RequestBody OrderRequest request)
```

Validation failures MUST produce:

* BusinessException
* HTTP 400
* No stacktrace exposure

---

# 8.7 Authentication & Authorization Failures

Authentication failures:

* MUST return HTTP 401
* MUST NOT reveal whether username exists

Authorization failures:

* MUST return HTTP 403
* MUST NOT expose permission model

Forbidden:

```json
{
  "message": "User does not have ROLE_ADMIN required by method X"
}
```

Allowed:

```json
{
  "errorCode": "ACCESS_DENIED",
  "message": "Access denied"
}
```

---

# 8.8 Security Logging Requirements

Security-related exceptions MUST:

* Be logged at ERROR
* Include correlationId
* Include userId (if authenticated)
* Exclude sensitive payload fields

Example:

```java id="sec01"
log.error("ACCESS_DENIED user={} correlationId={}",
          userId,
          MDC.get("correlationId"));
```

---

# 8.9 Information Leakage Risk Matrix

| Risk                | Example                       | Severity |
| ------------------- | ----------------------------- | -------- |
| SQL exposure        | Table name leaked             | High     |
| Stacktrace exposure | Class path leaked             | High     |
| Token leak          | Authorization header logged   | Critical |
| Internal host leak  | DB host returned              | High     |
| Business rule leak  | Fraud detection logic exposed | Medium   |

---

# 8.10 Secure Error Code Registry

Error codes MUST NOT:

* Reveal internal system structure
* Expose vendor technology

Forbidden:

* POSTGRES_TIMEOUT
* ORACLE_SEQUENCE_ERROR

Allowed:

* DATABASE_UNAVAILABLE
* DATA_ACCESS_ERROR

---

# 8.11 Production Hardening Checklist

Before production deployment, verify:

* Stacktrace disabled in responses
* Log sanitization active
* Sensitive headers masked
* Error mapping JSON validated
* Security exception mapping tested
* No internal class names in responses
* Correlation ID always returned

---

# 8.12 Anti-Patterns (Strictly Forbidden)

* Returning `e.getMessage()` directly
* Logging full request body without sanitization
* Exposing SQL in error message
* Including raw exception in API response
* Sending internal stacktrace to client
* Embedding credentials in logs

---

# 8.13 Security Escalation Rule

SystemCriticalException related to:

* Data corruption
* Integrity violation
* Authentication bypass
* Authorization bypass

MUST:

* Trigger alert
* Be logged at ERROR
* Be monitored by security team

---

# 8.14 Audit & Compliance Requirements

Security logging must allow:

* Traceability (who did what)
* Incident reconstruction
* Correlation across services
* Time-based forensic analysis

CorrelationId is mandatory in every error response.

---

# 8.15 Summary

Security requirements for exception handling:

* No information leakage
* Strict separation domain/transport
* Sanitized messages
* Sensitive data masking
* No stacktrace exposure
* Controlled authentication errors
* Alerting for critical failures
* Auditability guaranteed

Failure to comply constitutes a security breach risk.

---

If you want, next we can:

* Align this with OWASP Top 10 mapping formally
* Add compliance annotations for ISO 27001 / PCI DSS
* Or produce a final consolidated enterprise document v1.1

You are now at banking-grade security level for exception management.


