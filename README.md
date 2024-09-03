Yes, the **ShedLock** solution is compatible with **Oracle** databases. ShedLock supports any relational database that can be accessed using Spring's `JdbcTemplate`, which includes Oracle.

### Steps to Use ShedLock with Oracle Database

To use ShedLock with an Oracle database, you need to ensure a few Oracle-specific configurations are in place. Here's how you can set it up:

#### 1. **Add ShedLock Dependencies**

Ensure you have added the ShedLock dependencies to your `pom.xml`:

```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>5.8.0</version> <!-- Use the latest version -->
</dependency>
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-provider-jdbc-template</artifactId>
    <version>5.8.0</version> <!-- Use the latest version -->
</dependency>
```

#### 2. **Create the ShedLock Table in Oracle**

Create the `shedlock` table in your Oracle database. Below is the SQL script tailored for Oracle:

```sql
CREATE TABLE shedlock (
    name VARCHAR2(64) NOT NULL PRIMARY KEY,  -- Lock name
    lock_until TIMESTAMP(3) NULL,            -- The time until which the lock is valid
    locked_at TIMESTAMP(3) NULL,             -- The time when the lock was obtained
    locked_by VARCHAR2(255) NULL             -- Information about the node that holds the lock
);
```

Run this SQL script in your Oracle database to create the necessary table.

#### 3. **Configure ShedLock in Spring Boot**

Define a configuration class to set up the `LockProvider` for ShedLock using `JdbcTemplate` for the Oracle database:

```java
import net.javacrumbs.shedlock.spring.annotation.EnableSchedulerLock;
import net.javacrumbs.shedlock.core.LockProvider;
import net.javacrumbs.shedlock.provider.jdbctemplate.JdbcTemplateLockProvider;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;

@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")  // Default maximum lock duration
public class ShedLockConfig {

    @Bean
    public LockProvider lockProvider(@Autowired JdbcTemplate jdbcTemplate) {
        return new JdbcTemplateLockProvider(
            JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(jdbcTemplate)
                .usingDbTime()  // Use DB server time to avoid clock differences between nodes
                .build()
        );
    }
}
```

#### 4. **Annotate Your Scheduled Tasks**

Use the `@SchedulerLock` annotation to lock your scheduled tasks:

```java
import net.javacrumbs.shedlock.spring.annotation.SchedulerLock;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class MyScheduledTask {

    @Scheduled(cron = "0 */2 * * * *")  // Run every 2 minutes
    @SchedulerLock(name = "MyScheduledTask_sendDataToApi", lockAtMostFor = "10m", lockAtLeastFor = "2m")
    public void sendDataToApi() {
        // Your logic to send data to the API
        System.out.println("Sending data to API from the pod holding the lock");
    }
}
```

#### 5. **Configure Oracle Database Connection in `application.yml`**

Ensure your Spring Boot application is configured correctly to connect to your Oracle database. Here is an example configuration for `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/ORCLPDB1  # Update with your Oracle database URL
    username: your_username
    password: your_password
    driver-class-name: oracle.jdbc.OracleDriver
  jpa:
    hibernate:
      ddl-auto: validate  # Use 'validate' or 'none' in production environments
    show-sql: true
```

Make sure you have the Oracle JDBC driver in your classpath. If not, you need to add it as a dependency in your project.

#### 6. **Deploy and Test Your Application**

Deploy your application to your environment and check if the scheduled task runs on only one instance (pod) at a time.

- You can monitor the `shedlock` table in your Oracle database to see which task is locked and which pod holds the lock.
- Ensure your Oracle database is properly configured for high availability and performance.

### Conclusion

Yes, the ShedLock solution is fully compatible with Oracle databases. It works smoothly with any database supported by Spring's `JdbcTemplate`, including Oracle. This approach allows you to implement distributed locks easily without setting up additional infrastructure like Redis or Zookeeper.

If you need more help with Oracle-specific configurations or ShedLock tuning, feel free to ask!
