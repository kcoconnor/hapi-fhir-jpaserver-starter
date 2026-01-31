# 11. Test Strategy with Surefire/Failsafe Separation

Date: 2026-01-31

## Status

Accepted

## Context

The HAPI FHIR JPA Server Starter requires comprehensive testing to ensure:

1. **Unit Tests** - Fast, isolated tests of individual components
2. **Integration Tests** - Full server startup tests with database and HTTP interactions
3. **Multi-Version Coverage** - Tests across FHIR versions (DSTU2, DSTU3, R4, R4B, R5)
4. **Database Variants** - Tests with different database backends (H2, PostgreSQL, Elasticsearch)
5. **CI/CD Integration** - Clear separation of fast and slow tests for pipeline optimization

Testing strategies considered:

- **Single Test Phase** - All tests run together; simple but slow feedback
- **Maven Surefire Only** - Unit tests only; misses integration scenarios
- **Maven Failsafe Separation** - Unit and integration tests in distinct phases
- **Gradle Test Suites** - Gradle's native test suite support
- **Testcontainers Only** - All tests use containers; consistent but slower

## Decision

We will use Maven Surefire for unit tests (`*Test.java`) and Maven Failsafe for integration tests (`*IT.java`), with Testcontainers providing database infrastructure for integration tests.

Maven configuration:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-failsafe-plugin</artifactId>
    <version>${maven.failsafe.version}</version>
    <configuration>
        <redirectTestOutputToFile>true</redirectTestOutputToFile>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>integration-test</goal>
                <goal>verify</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

Test file naming conventions:

| Pattern | Plugin | Phase | Examples |
|---------|--------|-------|----------|
| `*Test.java` | Surefire | test | `CustomInterceptorTest.java`, `CustomBeanTest.java` |
| `*IT.java` | Failsafe | integration-test | `ExampleServerR4IT.java`, `MultitenantServerR4IT.java` |

Integration test structure:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ExampleServerR4IT {
    @LocalServerPort
    private int port;

    @Test
    void testCreateAndReadPatient() {
        IGenericClient client = ctx.newRestfulGenericClient("http://localhost:" + port + "/fhir");
        // Full HTTP integration test
    }
}
```

Testcontainers for database testing:

```java
@Testcontainers
public class PostgresElasticsearchPatientIT {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Container
    static ElasticsearchContainer elastic = new ElasticsearchContainer("elasticsearch:8.x");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.elasticsearch.uris", elastic::getHttpHostAddress);
    }
}
```

Build commands:

```bash
# Unit tests only (fast)
mvn test

# Unit + integration tests (full verification)
mvn verify

# Skip all tests
mvn install -DskipTests

# Run single integration test
mvn verify -Dit.test=ExampleServerR4IT
```

## Consequences

### Positive

- **Fast Feedback Loop**: `mvn test` runs only unit tests for quick developer feedback (seconds)
- **Full Verification**: `mvn verify` runs complete test suite including integration tests
- **Clear Naming Convention**: `*Test` vs `*IT` suffix immediately indicates test type and expected runtime
- **CI Pipeline Optimization**: Can run unit tests on every commit, integration tests on merges
- **Database Flexibility**: Testcontainers enables testing against real PostgreSQL/Elasticsearch without manual setup
- **FHIR Version Coverage**: Separate IT classes for each version (ExampleServerR4IT, ExampleServerR5IT)
- **Output Management**: `redirectTestOutputToFile` keeps console clean during integration tests

### Negative

- **Longer Full Build**: Integration tests significantly increase full build time (minutes vs seconds)
- **Resource Requirements**: Testcontainers requires Docker; not all CI environments support this
- **Test Duplication**: Some logic may be tested in both unit and integration tests
- **H2/Production Gap**: Unit tests use H2; behavior differences may only appear in database-specific ITs
- **Complexity**: Two test phases with different plugins adds Maven configuration complexity

### Neutral

- Failsafe version is aligned with Surefire version via `${maven.failsafe.version}` property
- Integration tests use `@SpringBootTest` with `RANDOM_PORT` to avoid port conflicts
- Testcontainers dependencies are test-scoped to avoid production classpath pollution
- The `hapi-fhir-jpaserver-test-utilities` dependency provides test helpers from HAPI core
- Awaitility is available for asynchronous test assertions (subscription tests, batch jobs)
