# 10. Hierarchical YAML Configuration with Environment Override

Date: 2026-01-31

## Status

Accepted

## Context

The HAPI FHIR JPA Server has extensive configuration options spanning:

- Server settings (port, context path)
- Database configuration (connection, pooling, dialect)
- FHIR version and feature flags
- Clinical Reasoning settings
- Subscription configuration
- Search and indexing options
- Validation settings
- CORS configuration
- Logging configuration

The configuration system must support:

1. **Sensible Defaults** - Work out of the box without configuration
2. **Environment-Specific Overrides** - Different settings for dev, staging, production
3. **Hierarchical Organization** - Related settings grouped logically
4. **Multiple Override Sources** - File, environment variables, command line
5. **Type Safety** - Validated configuration with clear error messages

Configuration approaches considered:

- **Java Properties Files** - Flat namespace, limited typing
- **XML Configuration** - Verbose, type-safe with schema
- **YAML Configuration** - Hierarchical, readable, Spring Boot native
- **Environment Variables Only** - 12-factor compliant but verbose for complex config
- **External Config Service** - Spring Cloud Config, adds infrastructure dependency

## Decision

We will use Spring Boot's YAML configuration with a primary `application.yaml` file, type-safe binding via `@ConfigurationProperties`, and environment variable overrides following Spring Boot's relaxed binding rules.

Primary configuration structure (`src/main/resources/application.yaml`):

```yaml
server:
  port: 8080
  tomcat:
    relaxed-query-chars: "|"

spring:
  datasource:
    url: jdbc:h2:mem:test_mem
    username: sa
    driver-class-name: org.h2.Driver
    hikari:
      maximum-pool-size: 10

  jpa:
    properties:
      hibernate:
        dialect: ca.uhn.fhir.jpa.model.dialect.HapiFhirH2Dialect
        hbm2ddl:
          auto: update

hapi:
  fhir:
    fhir_version: R4
    openapi_enabled: true
    cr:
      enabled: false
    mdm_enabled: false
    cors:
      allow_Credentials: true
      allowed_origin:
        - "*"
    subscription:
      resthook_enabled: false
      websocket_enabled: false
```

Type-safe binding:

```java
@ConfigurationProperties(prefix = "hapi.fhir")
@Configuration
public class AppProperties {
    private FhirVersionEnum fhir_version = FhirVersionEnum.R4;
    private Boolean openapi_enabled = false;
    private Cors cors = null;
    private Subscription subscription = new Subscription();

    public static class Cors {
        private Boolean allow_Credentials = true;
        private List<String> allowed_origin = List.of("*");
    }

    public static class Subscription {
        private Boolean resthook_enabled = false;
        private Email email = null;
    }
}
```

Environment variable override examples:

```bash
# Override database URL
SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/hapi

# Override FHIR version
HAPI_FHIR_FHIR_VERSION=R5

# Enable MDM
HAPI_FHIR_MDM_ENABLED=true

# Configure CORS origins
HAPI_FHIR_CORS_ALLOWED_ORIGIN_0=https://app.example.com
```

## Consequences

### Positive

- **Readable Structure**: YAML hierarchy mirrors logical configuration groupings (spring.datasource, hapi.fhir.cr)
- **Spring Boot Native**: Full support for Spring Boot's configuration precedence and relaxed binding
- **Environment Variable Override**: Any property can be overridden via environment variable without code changes
- **Type-Safe Access**: `AppProperties` provides compile-time checking and IDE autocomplete
- **Default Values**: Field initializers provide sensible defaults that YAML can override
- **Comments as Documentation**: YAML supports inline comments documenting configuration options
- **Profile Support**: `application-{profile}.yaml` enables environment-specific configuration

### Negative

- **YAML Sensitivity**: Indentation errors cause parse failures; can be frustrating to debug
- **Property Name Translation**: Environment variables use SCREAMING_SNAKE_CASE while YAML uses kebab-case/snake_case
- **Nested Structure Complexity**: Deeply nested properties have long environment variable names
- **List Override Limitations**: Overriding list items via environment variables requires index notation
- **Documentation Drift**: Comments in YAML may become stale as code evolves
- **Type Coercion**: Some type conversions (e.g., enum values) require exact matching

### Neutral

- Maven resource filtering with `@` delimiter substitutes build properties (e.g., `@project.artifactId@`)
- The `YamlPropertySourceLoader` bean enables programmatic YAML loading if needed
- Configuration validation via `@Validated` on `AppProperties` is available but not extensively used
- Spring Boot's configuration metadata (spring-configuration-metadata.json) could provide IDE support but is not generated
