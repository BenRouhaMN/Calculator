Perfect.
We’ll now produce the **final, clean, copy-ready version**, including:

* ✅ `PlatformException` as a **sealed class**
* ✅ Clear exception hierarchy
* ✅ Hybrid `HttpErrorMapper` (override + family fallback)
* ✅ No HTTP leakage into core
* ✅ No magic constants
* ✅ Proper formatted logging
* ✅ Fully structured and consistent

This version is **production-ready baseline for a banking platform foundation**.

---

# 📌 Exception Handling Standard – Final Version

---

# 1️⃣ ErrorCode (Core Layer)

## Purpose

Centralized catalog of all platform error codes.

Responsibilities:

* Provide stable external error identifier
* Provide localization key
* Indicate retryability
* Remain transport-agnostic

---

```java
package com.example.platform.error;

public enum ErrorCode {

    // ===== BUSINESS =====
    INVALID_TRADE_PRICE(
            "TRD-001",
            "error.trade.invalid.price",
            false
    ),

    INSUFFICIENT_ACCOUNT_BALANCE(
            "ACC-001",
            "error.account.insufficient.balance",
            false
    ),

    // ===== TECHNICAL =====
    DATABASE_TIMEOUT(
            "TEC-001",
            "error.database.timeout",
            true
    ),

    MESSAGE_BROKER_UNAVAILABLE(
            "TEC-002",
            "error.message.broker.unavailable",
            true
    ),

    // ===== SYSTEM =====
    DATA_INTEGRITY_VIOLATION(
            "SYS-001",
            "error.data.integrity.violation",
            false
    ),

    CONFIGURATION_CORRUPTED(
            "SYS-002",
            "error.configuration.corrupted",
            false
    ),

    UNEXPECTED_ERROR(
            "SYS-999",
            "error.unexpected",
            false
    );

    private final String code;
    private final String messageKey;
    private final boolean retryable;

    ErrorCode(String code, String messageKey, boolean retryable) {
        this.code = code;
        this.messageKey = messageKey;
        this.retryable = retryable;
    }

    public String getCode() { return code; }
    public String getMessageKey() { return messageKey; }
    public boolean isRetryable() { return retryable; }
}
```

---

# 2️⃣ PlatformException (Sealed Base Class)

## Purpose

Root of the exception hierarchy.

Responsibilities:

* Carry ErrorCode
* Carry formatting arguments
* Support chaining
* Enforce controlled subclassing (sealed)
* Guarantee transactional rollback (RuntimeException)

---

```java
package com.example.platform.error;

public sealed abstract class PlatformException
        extends RuntimeException
        permits BusinessException,
                TechnicalException,
                SystemCriticalException {

    private final ErrorCode errorCode;
    private final Object[] args;

    protected PlatformException(ErrorCode errorCode, Object... args) {
        super(errorCode.name());
        this.errorCode = errorCode;
        this.args = args;
    }

    protected PlatformException(ErrorCode errorCode, Throwable cause, Object... args) {
        super(errorCode.name(), cause);
        this.errorCode = errorCode;
        this.args = args;
    }

    public ErrorCode getErrorCode() { return errorCode; }
    public Object[] getArgs() { return args; }
    public boolean isRetryable() { return errorCode.isRetryable(); }
}
```

Why sealed?

* Prevents uncontrolled extension
* Keeps the taxonomy stable
* Enforces architectural discipline

---

# 3️⃣ BusinessException

## Used for:

* Domain validation failures
* Functional rule violations
* Client errors

Not retryable.

---

```java
package com.example.platform.error;

public non-sealed class BusinessException extends PlatformException {

    public BusinessException(ErrorCode code, Object... args) {
        super(code, args);
    }
}
```

### Example

```java
public class InvalidTradePriceException extends BusinessException {

    public InvalidTradePriceException(double price) {
        super(ErrorCode.INVALID_TRADE_PRICE, price);
    }
}
```

---

# 4️⃣ TechnicalException

## Used for:

* Infrastructure failures
* Network issues
* Database timeout
* Broker unavailability

Usually retryable.

---

```java
package com.example.platform.error;

public non-sealed class TechnicalException extends PlatformException {

    public TechnicalException(ErrorCode code, Throwable cause) {
        super(code, cause);
    }
}
```

### Example

```java
public class DatabaseTimeoutException extends TechnicalException {

    public DatabaseTimeoutException(Throwable cause) {
        super(ErrorCode.DATABASE_TIMEOUT, cause);
    }
}
```

---

# 5️⃣ SystemCriticalException

## Used for:

* Data corruption
* Internal inconsistency
* Broken invariants

Not retryable. Should trigger alerting.

---

```java
package com.example.platform.error;

public non-sealed class SystemCriticalException extends PlatformException {

    public SystemCriticalException(ErrorCode code, Throwable cause) {
        super(code, cause);
    }
}
```

### Example

```java
public class DataIntegrityViolationException extends SystemCriticalException {

    public DataIntegrityViolationException(Throwable cause) {
        super(ErrorCode.DATA_INTEGRITY_VIOLATION, cause);
    }
}
```

---

# 6️⃣ HTTP Error Mapper (Hybrid Strategy)

## Strategy

Resolution order:

1. Per-ErrorCode override
2. Default mapping by exception family
3. Safe fallback (500)

No fragile switch on enum.

---

```java
package com.example.api.error;

import com.example.platform.error.*;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.util.EnumMap;
import java.util.Map;

@Component
public class HttpErrorMapper {

    private final Map<ErrorCode, HttpStatus> overrides;

    public HttpErrorMapper() {

        Map<ErrorCode, HttpStatus> map =
                new EnumMap<>(ErrorCode.class);

        // Example override
        map.put(ErrorCode.INVALID_TRADE_PRICE,
                HttpStatus.UNPROCESSABLE_ENTITY);

        this.overrides = Map.copyOf(map);
    }

    public HttpStatus resolve(PlatformException ex) {

        // 1. Per-code override
        HttpStatus overridden = overrides.get(ex.getErrorCode());
        if (overridden != null) {
            return overridden;
        }

        // 2. Default mapping by family
        return switch (ex) {
            case BusinessException be ->
                    HttpStatus.BAD_REQUEST;

            case TechnicalException te ->
                    HttpStatus.SERVICE_UNAVAILABLE;

            case SystemCriticalException sce ->
                    HttpStatus.INTERNAL_SERVER_ERROR;
        };
    }
}
```

---

# 7️⃣ ErrorResponse DTO

```java
package com.example.api.error;

import java.time.Instant;

public record ErrorResponse(
        Instant timestamp,
        String errorCode,
        String message,
        boolean retryable,
        String correlationId,
        String path
) {}
```

---

# 8️⃣ GlobalExceptionHandler

## Responsibilities

* Resolve localized message
* Log formatted message
* Log stacktrace
* Use ErrorCode for fallback
* Avoid magic constants

---

```java
package com.example.api.error;

import com.example.platform.error.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.*;
import org.springframework.context.MessageSource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.Locale;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log =
            LoggerFactory.getLogger(GlobalExceptionHandler.class);

    private final HttpErrorMapper httpErrorMapper;
    private final MessageSource messageSource;

    public GlobalExceptionHandler(HttpErrorMapper mapper,
                                  MessageSource messageSource) {
        this.httpErrorMapper = mapper;
        this.messageSource = messageSource;
    }

    @ExceptionHandler(PlatformException.class)
    public ResponseEntity<ErrorResponse> handlePlatform(
            PlatformException ex,
            Locale locale,
            HttpServletRequest request) {

        String formattedMessage =
                messageSource.getMessage(
                        ex.getErrorCode().getMessageKey(),
                        ex.getArgs(),
                        locale
                );

        log.error(
                "ErrorCode={}, retryable={}, message={}",
                ex.getErrorCode().getCode(),
                ex.isRetryable(),
                formattedMessage,
                ex
        );

        ErrorResponse response = new ErrorResponse(
                Instant.now(),
                ex.getErrorCode().getCode(),
                formattedMessage,
                ex.isRetryable(),
                MDC.get("correlationId"),
                request.getRequestURI()
        );

        return ResponseEntity
                .status(httpErrorMapper.resolve(ex))
                .body(response);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(
            Exception ex,
            Locale locale,
            HttpServletRequest request) {

        ErrorCode fallback = ErrorCode.UNEXPECTED_ERROR;

        String formattedMessage =
                messageSource.getMessage(
                        fallback.getMessageKey(),
                        null,
                        locale
                );

        log.error(
                "ErrorCode={}, message={}",
                fallback.getCode(),
                formattedMessage,
                ex
        );

        ErrorResponse response = new ErrorResponse(
                Instant.now(),
                fallback.getCode(),
                formattedMessage,
                fallback.isRetryable(),
                MDC.get("correlationId"),
                request.getRequestURI()
        );

        return ResponseEntity
                .internalServerError()
                .body(response);
    }
}
```

---

# 9️⃣ messages.properties

```properties
error.trade.invalid.price=Provided trade price {0} is invalid.
error.account.insufficient.balance=Account {0} has insufficient balance.
error.database.timeout=Database is temporarily unavailable.
error.message.broker.unavailable=Message broker is currently unavailable.
error.data.integrity.violation=Internal system inconsistency detected.
error.configuration.corrupted=System configuration is corrupted.
error.unexpected=An unexpected internal error occurred.
```

---

# 🎯 Final Architecture Characteristics

✔ Sealed exception hierarchy
✔ Controlled taxonomy
✔ Hybrid HTTP mapping
✔ No transport leakage in core
✔ No magic constants
✔ Formatted logging
✔ Safe fallback
✔ Bank-grade baseline

---

If you want, next step can be:

* Making ErrorCode exhaustive via switch validation
* Adding metrics per ErrorCode
* Adding structured logging policy
* Adding validation exception support
* Adding resilience integration (Retry / Circuit Breaker)
