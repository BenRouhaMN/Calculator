Voici une **Version 3 — complète et organisationnelle**.
Cette version inclut tout ce qui est généralement présent dans les **engineering handbooks de grandes plateformes (Stripe / Uber / Shopify / Spotify)**.

Elle couvre :

* philosophie et principes
* stratégies de test
* architecture testable
* test doubles
* test data strategy
* determinism
* edge cases
* CI enforcement
* mutation testing
* flaky test policy
* test performance budgets
* legacy code strategy
* maintainability
* review checklist
* organization-wide governance

Le format reste **Markdown prêt à être stocké dans Git**.

---

# Unit Testing Engineering Standard

Version: 3.0
Owner: Engineering Leadership
Applies to: All backend services and shared libraries

---

# 1. Purpose

This document defines the **mandatory engineering standard for unit testing across the organization**.

The goals are to ensure that unit tests are:

* deterministic
* reliable
* maintainable
* readable
* fast
* independent of infrastructure
* consistent across repositories

Well-designed unit tests enable:

* safe refactoring
* rapid feedback loops
* lower defect rates
* improved developer productivity

Engineering studies consistently show that disciplined testing practices reduce production defects by **30–50%** and significantly improve delivery velocity.

---

# 2. Scope

This standard applies to testing:

| Component            | Covered |
| -------------------- | ------- |
| Business services    | ✓       |
| Domain logic         | ✓       |
| Validation rules     | ✓       |
| Calculations         | ✓       |
| Mapping logic        | ✓       |
| Utility classes      | ✓       |
| Data transformations | ✓       |

This standard **does not apply to**:

| Test Type         | Description                             |
| ----------------- | --------------------------------------- |
| Integration tests | Database or infrastructure interactions |
| Contract tests    | API compatibility testing               |
| End-to-end tests  | Full system workflows                   |

Separate standards define these test types.

---

# 3. Definition of a Unit Test

A **unit test validates a small unit of behavior in isolation**.

A unit may be:

* a method
* a class
* a domain rule
* a transformation function
* a validator

A unit test must **not depend on real infrastructure**.

External dependencies must be replaced by **test doubles**.

Examples of dependencies that must be isolated:

* databases
* HTTP clients
* message brokers
* file systems
* external APIs

---

# 4. Characteristics of High-Quality Unit Tests

Unit tests must follow the **FIRST principles**.

| Principle       | Description                  |
| --------------- | ---------------------------- |
| Fast            | Execution in milliseconds    |
| Independent     | No dependency between tests  |
| Repeatable      | Same result every run        |
| Self-validating | Clear pass/fail outcome      |
| Timely          | Written with production code |

---

# 5. Testing Philosophy

The goal of unit testing is **to verify behavior**, not to test implementation details.

Focus on testing:

| Priority | Component               |
| -------- | ----------------------- |
| High     | Business rules          |
| High     | Domain logic            |
| High     | Calculations            |
| High     | Validation logic        |
| Medium   | Mapping logic           |
| Medium   | Data transformations    |
| Low      | trivial getters/setters |

Avoid testing:

* framework behavior
* generated code
* trivial data containers

---

# 6. Verification Strategies

Unit tests validate behavior through three strategies.

---

# 6.1 State Verification

Checks resulting **state or returned value**.

Example:

```java
int result = calculator.applyDiscount(100);

assertEquals(90,result);
```

Preferred strategy.

---

# 6.2 Behaviour Verification

Checks **interaction with dependencies**.

Example:

```java
verify(repository).save(order);
```

Used for orchestration logic.

---

# 6.3 Output Verification

Validates **transformed outputs**.

Example:

```java
UserDto dto = mapper.map(entity);

assertEquals("John", dto.name());
```

---

# 7. Choosing the Correct Strategy

| Scenario              | Strategy  |
| --------------------- | --------- |
| Pure computation      | State     |
| Transformation        | Output    |
| Service orchestration | Behaviour |

Default rule:

Prefer **state or output verification**.

---

# 8. AAA Test Structure

Tests must follow:

```
Arrange
Act
Assert
```

Example:

```java
@Test
void shouldCalculateTotal(){

    // Arrange
    Calculator calculator = new Calculator();

    // Act
    int result = calculator.sum(10,20);

    // Assert
    assertEquals(30,result);
}
```

Rule:

A test must contain **one Act phase only**.

---

# 9. Naming Conventions

Tests must describe behavior.

Accepted formats:

### Format A

```
should_<result>_when_<condition>
```

Example:

```
shouldReturnUser_whenUserExists
```

### Format B

```
given_<condition>_when_<action>_then_<result>
```

Example:

```
givenInvalidOrder_whenProcessing_thenExceptionThrown
```

---

# 10. Test Class Structure

Naming convention:

```
<ClassName>Test
```

Example:

```
OrderServiceTest
PaymentValidatorTest
```

Example structure:

```java
class OrderServiceTest {

    private OrderService service;

    @BeforeEach
    void setup(){
        service = new OrderService();
    }

}
```

---

# 11. Test Isolation

Tests must be **fully independent**.

Rules:

* no shared mutable state
* tests must run in any order
* each test creates its own data

Forbidden patterns:

* shared static fields
* test execution ordering
* global caches

---

# 12. Test Doubles Strategy

Unit tests replace external dependencies using **test doubles**.

| Type  | Purpose                   |
| ----- | ------------------------- |
| Dummy | placeholder values        |
| Stub  | returns predefined data   |
| Mock  | verifies interactions     |
| Fake  | simplified implementation |

Example usage:

| Dependency  | Replacement |
| ----------- | ----------- |
| Repository  | Mock        |
| HTTP client | Stub        |
| Cache       | Fake        |

---

# 13. Mocking Rules

Mock **only system boundaries**.

Examples:

Mock:

* database repositories
* HTTP clients
* message brokers
* external SDKs

Do not mock:

* domain entities
* value objects
* collections
* pure functions
* mappers

Over-mocking creates brittle tests.

---

# 14. Test Data Strategy

Tests must use **readable and minimal data**.

Recommended approaches:

### Test Data Builders

```java
Order order = new OrderBuilder()
    .withAmount(200)
    .build();
```

### Object Mothers

```java
Order order = OrderMother.validOrder();
```

---

# 15. Deterministic Testing

Tests must control non-deterministic dependencies.

Avoid direct usage of:

```
LocalDate.now()
Math.random()
System.currentTimeMillis()
UUID.randomUUID()
```

Use abstractions:

```
Clock
RandomProvider
UUIDProvider
```

---

# 16. Exception Testing

Exceptions must be explicitly verified.

Example:

```java
assertThrows(
    IllegalArgumentException.class,
    () -> service.process(null)
);
```

Optional validation:

```java
assertThatThrownBy(...)
    .hasMessageContaining("order");
```

---

# 17. Designing Code for Testability

Code must be structured for testing.

Best practices:

### Dependency Injection

Dependencies must be injected through constructors.

### Pure Functions

Prefer functions without side effects.

### Single Responsibility

Classes should perform one responsibility.

### Clear Architecture Boundaries

```
Domain → pure logic
Application → orchestration
Infrastructure → IO
```

---

# 18. Edge Case Strategy

Tests must cover critical edge cases.

Examples:

| Case              | Example         |
| ----------------- | --------------- |
| Null values       | null input      |
| Boundary values   | 0, max          |
| Empty collections | empty list      |
| Invalid data      | malformed input |

---

# 19. Flaky Test Prevention

Flaky tests are forbidden.

Common causes:

| Cause            | Example           |
| ---------------- | ----------------- |
| Timing           | Thread.sleep      |
| Random values    | random generators |
| Concurrency      | race conditions   |
| External systems | network calls     |

Rules:

* no time dependency
* no randomness without control
* no concurrency

---

# 20. Test Maintainability

Tests must be easy to read and maintain.

Guidelines:

* tests should be **5–20 lines**
* avoid duplicated setup
* use builders for complex objects
* keep assertions explicit

Tests are **production code** and must follow the same quality standards.

---

# 21. Test Coverage Policy

Coverage targets:

| Metric          | Target |
| --------------- | ------ |
| Line coverage   | ≥ 80%  |
| Branch coverage | ≥ 75%  |

Coverage is a **quality indicator**, not the goal.

---

# 22. Mutation Testing (Advanced Quality Control)

Mutation testing validates the **effectiveness of tests**.

Tools:

* PIT mutation testing
* Stryker

Example rule:

```
Mutation score ≥ 60%
```

Mutation testing helps detect:

* weak assertions
* missing test scenarios

---

# 23. CI Enforcement

CI pipelines must enforce:

* all tests pass
* coverage thresholds respected
* no ignored tests
* no flaky tests

Recommended CI policies:

```
Coverage drop > 2% → build fails
```

Test execution time target:

```
Unit test suite < 60 seconds
```

---

# 24. Test Pyramid Strategy

Testing should follow the **Test Pyramid**.

```
        E2E
     Integration
    Unit Tests
```

Typical distribution:

| Test Type   | Target |
| ----------- | ------ |
| Unit        | 60–70% |
| Integration | 20–30% |
| E2E         | 5–10%  |

---

# 25. Legacy Code Testing Strategy

When testing legacy code:

1. Identify seams in the code
2. Introduce dependency injection
3. Write characterization tests
4. Gradually refactor

Never refactor legacy code without **test coverage first**.

---

# 26. Test Performance Budget

Unit tests must remain extremely fast.

Targets:

| Metric                | Target       |
| --------------------- | ------------ |
| Single test execution | < 10 ms      |
| Test suite            | < 60 seconds |

Slow tests must be migrated to integration tests.

---

# 27. Common Anti-Patterns

Avoid:

### Testing implementation details

Example:

```
verifying private methods
```

### Over-mocking

Mocking internal domain logic.

### Large tests

Tests covering multiple behaviors.

### Assertion-less tests

Tests without assertions.

---

# 28. Example Complete Test

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    PaymentGateway gateway;

    @InjectMocks
    PaymentService service;

    @Test
    void shouldProcessPayment_whenGatewayAccepts(){

        Payment payment = new Payment("123",100);

        when(gateway.charge(payment))
            .thenReturn(true);

        boolean result = service.process(payment);

        assertTrue(result);
    }
}
```

---

# 29. Pull Request Checklist

Before merging:

* unit tests added
* edge cases covered
* naming conventions respected
* CI pipeline green
* no flaky tests introduced

---

# 30. Mandatory Rules Summary

1. Tests must follow AAA structure
2. Tests must be deterministic
3. External dependencies must be replaced by test doubles
4. Prefer state verification over behavior verification
5. Tests must remain small and readable
6. Coverage ≥ 80%
7. CI must enforce test execution
8. Edge cases must be covered

---

💡 Si tu veux, je peux aussi te produire une **version encore plus avancée utilisée dans les grandes plateformes tech** avec :

* **architecture de tests par couche (hexagonal / clean architecture)**
* **patterns de test avancés**
* **organisation des packages de tests**
* **golden tests**
* **property-based testing**
* **contract tests internes**
* **testcontainers strategy**

Ce serait alors un **standard niveau staff/principal engineer utilisé dans les big tech**.
