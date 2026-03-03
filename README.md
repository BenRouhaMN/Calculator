
Parfait ✅
Voici la **version complète en format `.md`**, propre, prête à copier-coller dans un repository Git.

Tu peux créer un fichier :

```
exception-architecture.md
```

Et coller le contenu ci-dessous tel quel.

---

# Exception Architecture – Final Version

## Overview

This document defines the final exception architecture model.

Goals:

* Strong typing
* Deterministic behavior
* Clear separation of concerns
* Transport-agnostic error model
* Domain expressiveness
* i18n support
* Clean HTTP integration

---

# 1. Severity

```java
package com.example.platform.error;

public enum Severity {
    LOW,
    MEDIUM,
    HIGH,
    CRITICAL
}
```

---

# 2. ErrorCode (Transport-Agnostic)

```java
package com.example.platform.error;

public enum ErrorCode {

    // =========================
    // Business Errors
    // =========================
    INVALID_TRADE_PRICE(
            "BUS-001",
            "error.trade.invalid.price",
            Severity.LOW,
            false
    ),

    ORDER_ALREADY_EXISTS(
            "BUS-002",
            "error.order.already.exists",
            Severity.LOW,
            false
    ),

    // =========================
    // Technical Errors
    // =========================
    DATABASE_TIMEOUT(
            "TECH-001",
            "error.database.timeout",
            Severity.HIGH,
            true
    ),

    EXTERNAL_SERVICE_FAILURE(
            "TECH-002",
            "error.external.service.failure",
            Severity.HIGH,
            true
    ),

    // =========================
    // Critical Errors
    // =========================
    DATA_INTEGRITY_VIOLATION(
            "CRIT-001",
            "error.data.integrity.violation",
            Severity.CRITICAL,
            false
    );

    private final String code;
    private final String messageKey;
    private final Severity severity;
    private final boolean retryable;

    ErrorCode(String code,
              String messageKey,
              Severity severity,
              boolean retryable) {
        this.code = code;
        this.messageKey = messageKey;
        this.severity = severity;
        this.retryable = retryable;
    }

    public String getCode() { return code; }

    public String getMessageKey() { return messageKey; }

    public Severity getSeverity() { return severity; }

    public boolean isRetryable() { return retryable; }
}
```

---

# 3. Base PlatformException

```java
package com.example.platform.error;

public sealed abstract class PlatformException
        extends RuntimeException
        permits BusinessException,
                TechnicalException,
                SystemCriticalException {

    private final ErrorCode errorCode;
    private final Object[] args;

    protected PlatformException(ErrorCode errorCode,
                                Throwable cause,
                                Object... args) {
        super(cause);
        this.errorCode = errorCode;
        this.args = args;
    }

    public ErrorCode getErrorCode() { return errorCode; }

    public Object[] getArgs() { return args; }

    public boolean isRetryable() { return errorCode.isRetryable(); }

    public Severity getSeverity() { return errorCode.getSeverity(); }
}
```

---

# 4. Behavioral Families

## 4.1 BusinessException

```java
package com.example.platform.error;

public abstract sealed class BusinessException
        extends PlatformException
        permits InvalidTradePriceException,
                OrderAlreadyExistsException {

    protected BusinessException(ErrorCode errorCode,
                                Object... args) {
        super(errorCode, null, args);
    }
}
```

---

## 4.2 TechnicalException

```java
package com.example.platform.error;

public abstract sealed class TechnicalException
        extends PlatformException
        permits CsvFileCorruptedException {

    protected TechnicalException(ErrorCode errorCode,
                                 Throwable cause,
                                 Object... args) {
        super(errorCode, cause, args);
    }
}
```

---

## 4.3 SystemCriticalException

```java
package com.example.platform.error;

public abstract sealed class SystemCriticalException
        extends PlatformException
        permits DataIntegrityViolationException {

    protected SystemCriticalException(ErrorCode errorCode,
                                      Throwable cause,
                                      Object... args) {
        super(errorCode, cause, args);
    }
}
```

---

# 5. Domain-Specific Exceptions

## 5.1 Business Example

```java
package com.example.domain.trade;

import com.example.platform.error.*;
import java.math.BigDecimal;

public final class InvalidTradePriceException
        extends BusinessException {

    public InvalidTradePriceException(BigDecimal price) {
        super(ErrorCode.INVALID_TRADE_PRICE, price);
    }
}
```

---

## 5.2 Business Example

```java
public final class OrderAlreadyExistsException
        extends BusinessException {

    public OrderAlreadyExistsException(String orderId) {
        super(ErrorCode.ORDER_ALREADY_EXISTS, orderId);
    }
}
```

---

## 5.3 Technical Example

```java
package com.example.infrastructure.csv;

import com.example.platform.error.*;

public final class CsvFileCorruptedException
        extends TechnicalException {

    public CsvFileCorruptedException(Throwable cause) {
        super(ErrorCode.EXTERNAL_SERVICE_FAILURE, cause);
    }
}
```

---

## 5.4 Critical Example

```java
package com.example.platform.integrity;

import com.example.platform.error.*;

public final class DataIntegrityViolationException
        extends SystemCriticalException {

    public DataIntegrityViolationException(Throwable cause) {
        super(ErrorCode.DATA_INTEGRITY_VIOLATION, cause);
    }
}
```

---

# 6. HTTP Mapping (API Layer Only)

```java
package com.example.api.error;

import com.example.platform.error.ErrorCode;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

@Component
public class HttpErrorMapper {

    public HttpStatus resolve(ErrorCode code) {
        return switch (code) {

            case INVALID_TRADE_PRICE -> HttpStatus.BAD_REQUEST;
            case ORDER_ALREADY_EXISTS -> HttpStatus.CONFLICT;

            case DATABASE_TIMEOUT -> HttpStatus.SERVICE_UNAVAILABLE;
            case EXTERNAL_SERVICE_FAILURE -> HttpStatus.SERVICE_UNAVAILABLE;

            case DATA_INTEGRITY_VIOLATION -> HttpStatus.INTERNAL_SERVER_ERROR;
        };
    }
}
```

---

# 7. ErrorResponse DTO

```java
package com.example.api.error;

import java.time.Instant;

public record ErrorResponse(
        Instant timestamp,
        String errorCode,
        String message,
        boolean retryable,
        String correlationId
) {}
```

---

# 8. GlobalExceptionHandler

```java
package com.example.api.error;

import com.example.platform.error.PlatformException;
import org.slf4j.MDC;
import org.springframework.context.MessageSource;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.Locale;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private final HttpErrorMapper httpErrorMapper;
    private final MessageSource messageSource;

    public GlobalExceptionHandler(HttpErrorMapper httpErrorMapper,
                                  MessageSource messageSource) {
        this.httpErrorMapper = httpErrorMapper;
        this.messageSource = messageSource;
    }

    @ExceptionHandler(PlatformException.class)
    public ResponseEntity<ErrorResponse> handlePlatform(
            PlatformException ex,
            Locale locale) {

        String message = messageSource.getMessage(
                ex.getErrorCode().getMessageKey(),
                ex.getArgs(),
                locale
        );

        ErrorResponse response = new ErrorResponse(
                Instant.now(),
                ex.getErrorCode().getCode(),
                message,
                ex.isRetryable(),
                MDC.get("correlationId")
        );

        return ResponseEntity
                .status(httpErrorMapper.resolve(ex.getErrorCode()))
                .body(response);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex) {

        ErrorResponse response = new ErrorResponse(
                Instant.now(),
                "GEN-000",
                "Unexpected internal error",
                false,
                MDC.get("correlationId")
        );

        return ResponseEntity
                .internalServerError()
                .body(response);
    }
}
```

---

# 9. messages.properties

```properties
error.trade.invalid.price=Provided trade price {0} is invalid.
error.order.already.exists=Order with id {0} already exists.
error.database.timeout=Database is temporarily unavailable.
error.external.service.failure=External service failed.
error.data.integrity.violation=Internal system inconsistency detected.
```

---

# Final Characteristics

✔ Strong typing
✔ Domain expressive
✔ Behavior deterministic
✔ Transport-agnostic
✔ i18n-ready
✔ Retry-aware
✔ Clean separation of concerns
✔ No runtime JSON mapping
✔ No instanceof
✔ Enterprise-grade

---

Si tu veux, je peux maintenant te générer :

* 📦 Une structure Maven complète
* 🧪 Les tests unitaires associés
* 📊 Une version documentée pour architecture board
* 🔄 L’intégration retry & metrics

On stabilise ou on pousse encore plus loin ?
