# 4. Conditional Bean Loading for Multi-FHIR Version Support

Date: 2026-01-31

## Status

Accepted

## Context

HAPI FHIR supports multiple FHIR specification versions (DSTU2, DSTU3, R4, R4B, R5), each with different resource definitions, validation rules, and capabilities. The server starter must:

1. **Support All Versions** - Single codebase serving any FHIR version
2. **Runtime Selection** - Version chosen via configuration, not compile-time
3. **Avoid Classpath Conflicts** - Version-specific classes must not conflict
4. **Minimize Memory Usage** - Only load beans relevant to the configured version
5. **Enable Version-Specific Features** - Some features only apply to certain versions (e.g., IPS for R4+)

Approaches considered:

- **Separate Deployments per Version** - Simple but requires multiple builds and deployments
- **Runtime Class Loading** - Dynamic loading based on version, but complex and error-prone
- **Spring Conditional Beans** - Leverage Spring's condition mechanism for selective bean creation
- **Spring Profiles** - Use profile activation for version selection

## Decision

We will use Spring's `@Conditional` annotation with custom condition classes that evaluate the `hapi.fhir.fhir_version` configuration property at runtime.

Implementation structure:

**Condition Classes** (`ca.uhn.fhir.jpa.starter.annotations`):

```java
public class OnR4Condition implements Condition {
    @Override
    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata metadata) {
        FhirVersionEnum version = FhirVersionEnum.forVersionString(
            conditionContext.getEnvironment()
                .getProperty("hapi.fhir.fhir_version")
                .toUpperCase());
        return version == FhirVersionEnum.R4;
    }
}
```

Similar conditions exist for: `OnDSTU2Condition`, `OnDSTU3Condition`, `OnR4BCondition`, `OnR5Condition`.

**Version-Specific Configuration** (`ca.uhn.fhir.jpa.starter.common`):

```java
@Configuration
@Conditional(OnR4Condition.class)
@Import({JpaR4Config.class, StarterJpaConfig.class, StarterCrR4Config.class, StarterIpsConfig.class})
public class FhirServerConfigR4 {}
```

**Combined Condition for Shared Beans**:

```java
@Bean
@Conditional(OnEitherVersion.class)
public ServletRegistrationBean hapiServletRegistration(RestfulServer restfulServer) {
    // Servlet registered for any valid FHIR version
}
```

## Consequences

### Positive

- **Single Deployment Artifact**: One WAR/JAR supports all FHIR versions, simplifying deployment and maintenance
- **Configuration-Driven**: Version selection via `hapi.fhir.fhir_version: R4` in YAML without code changes
- **Memory Efficiency**: Only version-relevant beans are instantiated; unused version classes remain unloaded
- **Type Safety**: Compile-time verification of condition classes; IDE support for navigation
- **Extensibility**: New versions (e.g., R6) can be added by creating new condition classes
- **Clear Separation**: Version-specific configurations are isolated in separate classes

### Negative

- **Startup Overhead**: Condition evaluation adds minor startup time as Spring checks each conditional bean
- **Debugging Complexity**: Understanding which beans are loaded requires tracing condition evaluation
- **Testing Burden**: Each FHIR version requires separate integration test suites (ExampleServerR4IT, ExampleServerR5IT, etc.)
- **Duplication Risk**: Version-specific configuration classes may duplicate similar logic
- **Configuration Errors**: Invalid `fhir_version` values cause startup failures rather than graceful degradation

### Neutral

- Condition classes follow a predictable naming pattern (`On{Version}Condition`)
- Version-specific configs import the corresponding HAPI FHIR JPA config (`JpaR4Config`, `JpaR5Config`)
- The `FhirVersionEnum` from HAPI FHIR core provides canonical version mapping
