Below is a single section you can directly add to your engineering/testing standards.
It contains the context, the rationale behind the design, and the full code for the base classes (Kafka + DB cleaner + PostgreSQL).


---

Integration Test Architecture

Context

Integration tests validate that multiple components of the application work correctly together (services, repositories, messaging infrastructure, etc.).
Unlike unit tests, they require a real application context and often external infrastructure such as messaging brokers or databases.

To ensure consistency and avoid duplicated configuration across tests, the project defines a shared integration testing architecture based on reusable base classes.

The infrastructure required by tests is provisioned using Testcontainers, allowing tests to run against real services such as Apache Kafka and PostgreSQL in an isolated and reproducible environment.


---

Design Approach

The integration test framework follows a layered base-class approach.

A generic base class defines the common test configuration, while specialized subclasses introduce infrastructure components such as Kafka or databases.

AbstractIntegrationTest
      │
      ├── KafkaIntegrationTest
      │
      └── DatabaseIntegrationTest
             │
             └── PostgresIntegrationTest

This design provides several benefits:

Consistency
All integration tests share the same Spring test configuration.

Separation of concerns
Infrastructure components are introduced only when required.

Performance
Tests only start the containers they actually need.

Flexibility
Infrastructure technologies can evolve without refactoring the entire test suite.

Test isolation
The database is cleaned between tests to guarantee deterministic results.



---

Implementation

Base Integration Test

The base class defines the minimal configuration required for all integration tests using Spring Boot.

@SpringBootTest
@ActiveProfiles("test")
@Testcontainers
public abstract class AbstractIntegrationTest {
}


---

Kafka Integration Test Base

Tests that require Kafka extend this class.
A Kafka broker is started automatically using Testcontainers.

public abstract class KafkaIntegrationTest extends AbstractIntegrationTest {

    @Container
    static KafkaContainer kafka =
            new KafkaContainer(
                DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
            );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add(
            "spring.kafka.bootstrap-servers",
            kafka::getBootstrapServers
        );
    }

}


---

Database Cleaner

To guarantee test isolation, the database state must be reset between tests.
This is achieved using a pluggable DatabaseCleaner interface.

public interface DatabaseCleaner {

    void clean();

}


---

Database Integration Test Base

The base class responsible for cleaning the database before each test.

public abstract class DatabaseIntegrationTest extends AbstractIntegrationTest {

    @Autowired(required = false)
    DatabaseCleaner databaseCleaner;

    @BeforeEach
    void cleanDatabase() {
        if (databaseCleaner != null) {
            databaseCleaner.clean();
        }
    }

}


---

PostgreSQL Integration Test Base

When the persistence layer relies on PostgreSQL, the database container is started automatically.

public abstract class PostgresIntegrationTest extends DatabaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void configureDatasource(DynamicPropertyRegistry registry) {

        registry.add(
            "spring.datasource.url",
            postgres::getJdbcUrl
        );

        registry.add(
            "spring.datasource.username",
            postgres::getUsername
        );

        registry.add(
            "spring.datasource.password",
            postgres::getPassword
        );

    }

}


---

PostgreSQL Database Cleaner

Example implementation used to reset the database state between tests.

@Component
public class PostgresDatabaseCleaner implements DatabaseCleaner {

    private final JdbcTemplate jdbcTemplate;

    public PostgresDatabaseCleaner(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public void clean() {

        jdbcTemplate.execute("TRUNCATE TABLE users CASCADE");
        jdbcTemplate.execute("TRUNCATE TABLE orders CASCADE");

    }

}


---

Example Usage

A test requiring both Spring context and PostgreSQL simply extends the corresponding base class.

class UserRepositoryIT extends PostgresIntegrationTest {

    @Test
    void shouldPersistUser() {
        // integration test logic
    }

}


---

This architecture ensures that integration tests remain consistent, maintainable, scalable, and independent of infrastructure choices while still allowing tests to run against real services.
