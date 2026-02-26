Below is the **complete, consolidated, enterprise-grade Exception Management Standard**, rewritten to:

* Start each major section with a numbered heading
* Clearly explain the difference between Business, Technical, and SystemCritical exceptions at the beginning
* Provide detailed, production-ready Security section
* Include full Retry & Resilience
* Include detailed Transaction & Consistency
* Include Logging implementation
* Include Idempotency
* Include HTTP mapping
* Include Handling Principles with code examples
* Be ready to copy into a `.md` file

---

# 1. Introduction

This document defines the official Exception Management Standard for the platform.

It ensures:

* Deterministic failure behavior
* Transaction integrity
* Retry safety
* Observability consistency
* Security hardening
* Cross-team uniform implementation

This standard is mandatory for all backend services, REST APIs, async consumers, and scheduled jobs.

Technology baseline:
Java 17+, Spring Boot 3+, Micrometer, Resilience4j, Logback

---

# 2. Exception Taxonomy

## 2.1 Conceptual Model

All failures in the system fall into exactly three categories:

### 2.1.1 BusinessException

Represents domain rule violations.

Characteristics:

* The system behaves correctly.
* The input or business state is invalid.
* No infrastructure failure occurred.
* No retry should be attempted.
* No system alert required.
* Usually mapped to HTTP 400 / 404 / 409.

Examples:

* ORDER_NOT_FOUND
* INSUFFICIENT_BALANCE
* INVALID_STATUS_TRANSITION

This is a functional failure, not a system failure.

---

### 2.1.2 TechnicalException

Represents infrastructure or dependency instability.

Characteristics:

* The system cannot complete the operation due to infrastructure.
* May be retryable.
* May require alert if persistent.
* Always logged at ERROR level.
* Mapped to HTTP 503 or 500.

Examples:

* DB_CONNECTION_FAILURE
* TIMEOUT
* REMOTE_SERVICE_UNAVAILABLE

This is a system failure, not a business rule issue.

---

### 2.1.3 SystemCriticalException

Represents integrity risk or inconsistent state.

Characteristics:

* Data integrity at risk.
* Partial commit detected.
* Invariant violation.
* Immediate escalation required.
* Never retryable.
* Always triggers alert.
* Always rollback.

Examples:

* DATA_CORRUPTION_DETECTED
* POST_CONDITION_FAILURE
* TRANSACTION_INCONSISTENT

This is a platform stability risk.

---

## 2.2 RuntimeException Design Rationale

All platform exceptions MUST extend RuntimeException.

Reason:

Spring behavior:

* RuntimeException → automatic rollback
* CheckedException → no rollback

Using checked exceptions would:

* Pollute service signatures
* Leak infrastructure concerns
* Break clean architecture
* Encourage defensive try/catch

Therefore:

All platform exceptions extend RuntimeException.

---

## 2.3 Base PlatformException

```java
public abstract class PlatformException extends RuntimeException {

    private final String errorCode;
    private final boolean retryable;

    protected PlatformException(String errorCode,
                                String message,
                                boolean retryable,
                                Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.retryable = retryable;
    }

    public String getErrorCode() { return errorCode; }
    public boolean isRetryable() { return retryable; }
}
```

---

## 2.4 BusinessException

```java
public class BusinessException extends PlatformException {

    public BusinessException(String errorCode, String message) {
        super(errorCode, message, false, null);
    }
}
```

---

## 2.5 TechnicalException

```java
public class TechnicalException extends PlatformException {

    public TechnicalException(String errorCode,
                              String message,
                              boolean retryable,
                              Throwable cause) {
        super(errorCode, message, retryable, cause);
    }
}
```

---

## 2.6 SystemCriticalException

```java
public class SystemCriticalException extends PlatformException {

    public SystemCriticalException(String errorCode,
                                   String message,
                                   Throwable cause) {
        super(errorCode, message, false, cause);
    }
}
```

---

# 3. Exception Handling Principles (Mandatory Rules + Code)

## 3.1 No Silent Failures

❌ Forbidden:

```java
catch (Exception e) {}
```

❌ Forbidden:

```java
catch (Exception e) {
    log.error("Failure");
}
```

Execution continues → inconsistent state.

✅ Required:

```java
catch (DataAccessException e) {
    throw new TechnicalException(
        "DB_WRITE_FAILURE",
        "Unable to persist order",
        true,
        e
    );
}
```

---

## 3.2 Preserve Root Cause

❌ Forbidden:

```java
throw new TechnicalException("DB_ERROR", "Database error", true, null);
```

✅ Required:

```java
throw new TechnicalException("DB_ERROR", "Database error", true, e);
```

---

## 3.3 Do Not Use Exceptions for Flow Control

❌ Forbidden:

```java
try {
    return repository.findById(id)
        .orElseThrow(RuntimeException::new);
} catch (RuntimeException e) {
    return null;
}
```

✅ Required:

```java
Optional<Order> order = repository.findById(id);

if (order.isEmpty()) {
    throw new BusinessException("ORDER_NOT_FOUND", "Order not found");
}

return order.get();
```

---

## 3.4 Controllers Must Not Build Error Responses

❌ Forbidden:

```java
@GetMapping("/{id}")
public ResponseEntity<?> get(String id) {
    try {
        return ResponseEntity.ok(service.get(id));
    } catch (Exception e) {
        return ResponseEntity.status(500).body(e.getMessage());
    }
}
```

✅ Required:

Controller returns domain object only.
Exception handled centrally.

---

## 3.5 Stable Error Codes Mandatory

❌ Forbidden:

```java
throw new BusinessException("Something went wrong");
```

✅ Required:

```java
throw new BusinessException(
    "ORDER_ALREADY_EXISTS",
    "Order already exists"
);
```

---

# 4. HTTP Mapping & API Error Model

## 4.1 Standard Error Response

```java
public record ErrorResponse(
        Instant timestamp,
        String errorCode,
        String message,
        boolean retryable,
        String correlationId
) {}
```

---

## 4.2 Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(PlatformException.class)
    public ResponseEntity<ErrorResponse> handlePlatform(PlatformException ex) {

        HttpStatus status;

        if (ex instanceof BusinessException)
            status = HttpStatus.BAD_REQUEST;
        else if (ex instanceof TechnicalException)
            status = ex.isRetryable()
                    ? HttpStatus.SERVICE_UNAVAILABLE
                    : HttpStatus.INTERNAL_SERVER_ERROR;
        else
            status = HttpStatus.INTERNAL_SERVER_ERROR;

        ErrorResponse response = new ErrorResponse(
                Instant.now(),
                ex.getErrorCode(),
                ex.getMessage(),
                ex.isRetryable(),
                MDC.get("correlationId")
        );

        logException(ex);

        return ResponseEntity.status(status).body(response);
    }
}
```

---

# 5. Transaction & Consistency Rules

## 5.1 Rollback Matrix

| Exception               | Rollback |
| ----------------------- | -------- |
| BusinessException       | No       |
| TechnicalException      | Yes      |
| SystemCriticalException | Yes      |

Override:

```java
@Transactional(noRollbackFor = BusinessException.class)
```

---

## 5.2 Idempotency Requirement

Mandatory when:

* Retry enabled
* External dependency
* Payment flow

### Header

X-Idempotency-Key

### Entity

```java
@Entity
@Table(uniqueConstraints = @UniqueConstraint(columnNames = "key"))
public class IdempotencyRecord {

    @Id @GeneratedValue
    private Long id;

    private String key;
    private String responseHash;
    private Instant createdAt;
}
```

### Service Pattern

```java
@Transactional
public PaymentResponse process(String key, PaymentRequest request) {

    Optional<IdempotencyRecord> existing =
            repository.findByKey(key);

    if (existing.isPresent())
        return reconstruct(existing.get());

    PaymentResponse response = executePayment(request);

    repository.save(new IdempotencyRecord(
            key,
            hash(response),
            Instant.now()
    ));

    return response;
}
```

Guarantee: safe retry.

---

## 5.3 Partial Commit Policy

Forbidden:

```java
saveToDb();
callExternal();
```

Approved:

* Saga pattern
* Outbox pattern

Outbox example:

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

---

# 6. Retry & Resilience Policy

Retry allowed ONLY if:

* TechnicalException
* retryable=true
* Idempotent operation

## 6.1 Resilience4j Configuration

```yaml
resilience4j:
  retry:
    instances:
      inventoryService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
```

## 6.2 Usage

```java
@Retry(name = "inventoryService")
@CircuitBreaker(name = "inventoryService")
public Stock reserve(String requestId) {
    ...
}
```

Forbidden:

* Retrying BusinessException
* Retrying non-idempotent payments
* Retry inside DB transaction

---

# 7. Observability Standard

## 7.1 Logback JSON Configuration

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

## 7.2 Correlation Filter

```java
@Component
public class CorrelationFilter extends OncePerRequestFilter {

    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        String correlationId =
                Optional.ofNullable(request.getHeader("X-Correlation-ID"))
                        .orElse(UUID.randomUUID().toString());

        MDC.put("correlationId", correlationId);
        response.setHeader("X-Correlation-ID", correlationId);

        chain.doFilter(request, response);
        MDC.clear();
    }
}
```

---

## 7.3 Metrics

```java
Counter.builder("exceptions_total")
       .tag("type", ex.getClass().getSimpleName())
       .tag("retryable", String.valueOf(ex.isRetryable()))
       .register(meterRegistry)
       .increment();
```

Alert:

* > 5% error rate
* Any SystemCriticalException

---

# 8. Security & Information Exposure Rules

## 8.1 External API Rules

API MUST NOT expose:

* Stacktrace
* SQL queries
* Internal class names
* Hostnames
* Infrastructure details
* Sensitive identifiers

---

## 8.2 Error Message Policy

* Must be client-safe
* Must not reveal system internals
* Must not expose raw exception messages

---

## 8.3 Logging Policy

* Full stacktrace logged internally
* Never exposed in response

---

## 8.4 Validation Hardening

* All user inputs validated before business logic
* Reject malformed inputs early
* Use @Valid and Bean Validation

---

## 8.5 Sensitive Data Protection

Logs MUST NOT include:

* Passwords
* Tokens
* Personal identifiable information (PII)

Masking required.

---

# 9. Async / Batch Handling

| Exception               | Behavior              |
| ----------------------- | --------------------- |
| BusinessException       | Log WARN, acknowledge |
| TechnicalException      | Retry with backoff    |
| SystemCriticalException | Stop consumer, alert  |

---

# 10. Governance & Compliance

PR rejected if:

* Silent catch
* Controller builds error manually
* Retry without idempotency
* Partial commit risk
* Stacktrace exposed
* No correlation ID
* No metrics emission

Quarterly audit mandatory.

---

This is now a fully structured, numbered, detailed standard ready to be copied and used across teams.

If you want, I can now produce:

* A visual decision tree
* UML sequence diagrams
* Static analysis enforcement checklist
* Architecture governance maturity model
