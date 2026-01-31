# 2. Spring Boot as Core Framework

Date: 2026-01-31

## Status

Accepted

## Context

The HAPI FHIR JPA Server requires a robust application framework to provide:

1. **Dependency Injection** - Managing complex object graphs including DAOs, services, interceptors, and providers
2. **Configuration Management** - Externalized configuration with environment-specific overrides
3. **Web Server Integration** - Servlet container management for the FHIR REST API
4. **Database Connectivity** - JPA/Hibernate integration with connection pooling
5. **Production Readiness** - Health checks, metrics, and monitoring capabilities
6. **Developer Experience** - Rapid development cycle with hot reload capabilities

The HAPI FHIR core library uses Spring Framework internally for dependency injection. Several framework options were considered:

- **Plain Spring Framework** - Full control but requires significant boilerplate configuration
- **Spring Boot** - Opinionated defaults with auto-configuration, reducing boilerplate
- **Jakarta EE / Java EE** - Standard-based but less flexible configuration options
- **Quarkus** - Modern alternative but would require significant HAPI FHIR modifications
- **Micronaut** - Compile-time DI but incompatible with Spring-based HAPI FHIR core

## Decision

We will use Spring Boot 3.x as the core application framework, extending `SpringBootServletInitializer` to support both embedded server and traditional WAR deployment modes.

Key implementation details:

```java
@SpringBootApplication(exclude = {ThymeleafAutoConfiguration.class})
@Import({
    StarterCrR4Config.class,
    StarterCdsHooksConfig.class,
    SubscriptionProcessorConfig.class,
    MdmConfig.class,
    JpaBatch2Config.class,
    Batch2JobsConfig.class
})
public class Application extends SpringBootServletInitializer {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

The application leverages:
- Spring Boot Starter Web (with embedded Tomcat by default)
- Spring Boot Starter Data JPA (Hibernate integration)
- Spring Boot Starter Actuator (health endpoints, metrics)
- Spring Boot Configuration Properties (type-safe configuration)

## Consequences

### Positive

- **Reduced Configuration Complexity**: Auto-configuration handles most infrastructure concerns (DataSource, EntityManager, transaction management) with sensible defaults
- **Alignment with HAPI FHIR Core**: HAPI FHIR already uses Spring internally, ensuring seamless integration without adapter layers
- **Flexible Deployment**: `SpringBootServletInitializer` enables both embedded server (JAR) and traditional WAR deployment to external containers
- **Production-Ready Features**: Built-in actuator endpoints provide health checks (`/actuator/health`), metrics, and Prometheus integration out of the box
- **Large Ecosystem**: Extensive documentation, community support, and third-party integrations
- **Configuration Externalization**: Environment variables, YAML files, and profile-based configuration without code changes
- **Developer Productivity**: Hot reload with spring-boot-devtools, rapid startup in development mode

### Negative

- **Framework Lock-in**: Deep integration with Spring Boot makes migration to alternative frameworks costly
- **Opinionated Defaults**: Some auto-configuration may need explicit overrides for FHIR-specific requirements (e.g., Thymeleaf exclusion)
- **Memory Footprint**: Spring Boot applications typically have higher baseline memory consumption compared to microframework alternatives
- **Startup Time**: Cold start time is slower than frameworks with compile-time dependency injection (relevant for serverless deployments)
- **Transitive Dependencies**: Spring Boot brings a large dependency tree that may conflict with other libraries

### Neutral

- Version coupling means tracking Spring Boot release cycles for security updates
- Developers must understand both Spring Boot conventions and HAPI FHIR specifics
- Different Maven profiles (`boot`, `jetty`) manage embedded server selection
