# 8. Custom Bean/Interceptor/Provider Extension Points

Date: 2026-01-31

## Status

Accepted

## Context

Organizations customizing the HAPI FHIR JPA Server need to add:

1. **Custom Interceptors** - Security, auditing, transformation logic
2. **Custom Providers** - Additional FHIR operations, custom endpoints
3. **Custom Beans** - Services, repositories, configuration components
4. **Custom Resource Types** - Non-standard FHIR resources

The extension mechanism must:
- **Avoid Forking** - Customizations should not require modifying starter code
- **Support Dependency Injection** - Custom code should access Spring-managed beans
- **Be Configuration-Driven** - Specify custom classes via properties, not code
- **Enable Both Simple and Complex Cases** - POJOs for simple cases, Spring beans for complex ones

Approaches considered:

- **Code Modification** - Direct changes to starter classes; prevents upstream merges
- **Maven Overlays** - WAR overlay mechanism; complex build configuration
- **Service Loader** - Java SPI; requires META-INF/services files
- **Configuration Properties** - Specify class names in YAML; Spring instantiation

## Decision

We will provide three configuration-based extension points:

### 1. Custom Bean Packages

`@ComponentScan` dynamically includes user-specified packages:

```java
@Configuration
@ComponentScan(basePackages = {"${hapi.fhir.custom-bean-packages:}"})
public class StarterJpaConfig { }
```

Configuration:
```yaml
hapi:
  fhir:
    custom-bean-packages: com.myorg.fhir.custom
```

### 2. Custom Interceptor Classes

Registration via class name list:

```yaml
hapi:
  fhir:
    custom-interceptor-classes:
      - com.myorg.fhir.interceptor.AuditInterceptor
      - com.myorg.fhir.interceptor.ConsentInterceptor
```

Implementation:
```java
private void registerCustomInterceptors(RestfulServer fhirServer,
        ApplicationContext theAppContext, List<String> customInterceptorClasses) {
    for (String className : customInterceptorClasses) {
        Class clazz = Class.forName(className);

        // Try to get as Spring bean first
        Object interceptor = null;
        try {
            interceptor = theAppContext.getBean(clazz);
            ourLog.info("registering custom interceptor as bean: {}", className);
        } catch (NoSuchBeanDefinitionException ex) {
            // Not a bean, instantiate directly
        }

        // Fallback to reflection instantiation
        if (interceptor == null) {
            interceptor = clazz.getConstructor().newInstance();
            ourLog.info("registering custom interceptor as pojo: {}", className);
        }

        fhirServer.registerInterceptor(interceptor);
    }
}
```

### 3. Custom Provider Classes

Same pattern as interceptors:

```yaml
hapi:
  fhir:
    custom-provider-classes:
      - com.myorg.fhir.provider.CustomOperationProvider
```

Implementation follows the same bean-or-pojo pattern as interceptors.

## Consequences

### Positive

- **No Source Modification**: Customizations are purely configuration-based; easy to merge upstream updates
- **Spring Integration**: Custom classes in scanned packages become full Spring beans with DI support
- **Flexible Instantiation**: Classes can be either Spring beans (for dependency injection) or plain POJOs (for simple cases)
- **Explicit Registration**: Class names in configuration make customizations visible and auditable
- **Hot Deployment Compatible**: Some changes can be applied without rebuilding the application
- **Additive Model**: Customizations add to rather than replace default behavior

### Negative

- **Classpath Coupling**: Custom classes must be on the classpath at startup; no dynamic loading
- **String-Based Configuration**: Class names as strings lose IDE refactoring support and compile-time checking
- **No Ordering Control**: Custom interceptors are registered after built-in ones; order may matter
- **Limited Discovery**: No automatic scanning for custom interceptors/providers; must be explicitly listed
- **Startup Failures**: Invalid class names cause startup failure rather than graceful degradation

### Neutral

- The `@ComponentScan` basePackages supports multiple comma-separated packages
- Custom interceptors registered via config have access to the same hooks as built-in interceptors
- POJO instantiation requires a no-arg constructor; complex initialization requires Spring bean approach
- Custom providers must follow HAPI FHIR provider conventions (annotations, method signatures)
