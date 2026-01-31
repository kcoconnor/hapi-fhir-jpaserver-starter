# 6. H2 as Default Database with Multi-Database Support

Date: 2026-01-31

## Status

Accepted

## Context

The HAPI FHIR JPA Server requires a relational database for persisting FHIR resources. The starter project needs to:

1. **Work Out of the Box** - Zero configuration for initial exploration and development
2. **Support Production Databases** - PostgreSQL, SQL Server, Oracle for real deployments
3. **Enable Testing** - Consistent behavior in CI/CD pipelines
4. **Avoid External Dependencies** - No database installation required for getting started

Database options considered:

- **H2 In-Memory** - Zero setup, embedded, fast for development
- **H2 File-Based** - Persistence across restarts, still embedded
- **PostgreSQL** - Production-grade, widely used, HAPI-optimized dialect available
- **MySQL/MariaDB** - Popular but less optimal for HAPI's usage patterns
- **SQL Server** - Enterprise environments, Microsoft ecosystem
- **SQLite** - Lightweight but limited concurrent access

## Decision

We will use H2 database in in-memory mode as the default configuration, with explicit support for PostgreSQL and SQL Server as production database options.

Default configuration:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:test_mem
    username: sa
    password: null
    driver-class-name: org.h2.Driver
    hikari:
      maximum-pool-size: 10

  jpa:
    properties:
      hibernate:
        dialect: ca.uhn.fhir.jpa.model.dialect.HapiFhirH2Dialect
        hbm2ddl:
          auto: update
```

Production PostgreSQL configuration (commented in application.yaml):

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/hapi
    username: admin
    password: admin
    driver-class-name: org.postgresql.Driver
  jpa:
    properties:
      hibernate:
        dialect: ca.uhn.fhir.jpa.model.dialect.HapiFhirPostgresDialect
```

Database driver dependencies:

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
<dependency>
    <groupId>com.microsoft.sqlserver</groupId>
    <artifactId>mssql-jdbc</artifactId>
</dependency>
```

## Consequences

### Positive

- **Zero-Configuration Start**: `mvn spring-boot:run` works immediately without database setup
- **Fast Development Cycle**: In-memory H2 resets on restart, avoiding state pollution during development
- **Portable Testing**: Tests run consistently across environments without external database dependencies
- **HAPI-Optimized Dialects**: Custom Hibernate dialects (`HapiFhirH2Dialect`, `HapiFhirPostgresDialect`) optimize for HAPI FHIR's data patterns
- **Clear Upgrade Path**: Configuration-only switch to production database; no code changes required
- **Connection Pooling**: HikariCP included by default for efficient connection management

### Negative

- **Data Loss by Default**: In-memory H2 loses all data on restart; can surprise users expecting persistence
- **H2 Behavioral Differences**: H2's SQL dialect differs from production databases; queries may behave differently
- **Test/Production Parity Gap**: Tests pass on H2 but may fail on PostgreSQL due to SQL differences
- **Limited Concurrency**: H2 has limitations under high concurrent load that won't manifest until production
- **Schema Migration**: `hbm2ddl.auto: update` is convenient but not recommended for production schema management

### Neutral

- H2 file-based mode (`jdbc:h2:file:./target/database/h2`) is available but commented out
- PostgreSQL-specific features (JSONB, full-text search) are not used to maintain database portability
- Flyway database migrations are disabled by default (`spring.flyway.enabled: false`) but available
- The `cloudsql-postgres` Maven profile adds Google Cloud SQL socket factory for GCP deployments
