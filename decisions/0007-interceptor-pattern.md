# 7. HAPI Interceptor Pattern for Cross-Cutting Concerns

Date: 2026-01-31

## Status

Accepted

## Context

The FHIR server needs to handle cross-cutting concerns that span multiple resource types and operations:

- **Logging** - Request/response logging for auditing and debugging
- **Validation** - Request and response validation against FHIR profiles
- **Security** - Authentication, authorization, consent enforcement
- **CORS** - Cross-origin resource sharing for web clients
- **Binary Storage** - Handling large binary attachments
- **Cascading Deletes** - Deleting resources with dependencies
- **Subscriptions** - Triggering notifications on resource changes
- **Response Formatting** - HTML highlighting for browser clients

These concerns should be:
1. **Modular** - Independently deployable and testable
2. **Composable** - Multiple concerns can be combined
3. **Non-Invasive** - Don't require modifying core resource logic
4. **Configurable** - Easily enabled/disabled based on deployment needs

Approaches considered:

- **Servlet Filters** - Standard Java EE, but limited access to FHIR context
- **Spring AOP** - Aspect-oriented programming, but complex configuration
- **HAPI Interceptors** - Purpose-built for FHIR, rich hook point system
- **Custom Middleware** - Bespoke solution, high development cost

## Decision

We will use HAPI FHIR's interceptor framework, which provides a comprehensive set of hook points for the FHIR request lifecycle. Interceptors are registered with the `RestfulServer` during startup.

Interceptor registration in `StarterJpaConfig`:

```java
@Bean
public RestfulServer restfulServer(...) {
    RestfulServer fhirServer = new RestfulServer(fhirSystemDao.getContext());

    // Always-on interceptors
    fhirServer.registerInterceptor(new ResponseHighlighterInterceptor());
    fhirServer.registerInterceptor(loggingInterceptor);

    // Conditional interceptors based on configuration
    corsInterceptor.ifPresent(fhirServer::registerInterceptor);

    if (appProperties.getAllow_cascading_deletes()) {
        CascadingDeleteInterceptor cascadingDeleteInterceptor = new CascadingDeleteInterceptor(...);
        fhirServer.registerInterceptor(cascadingDeleteInterceptor);
    }

    if (appProperties.getBinary_storage_enabled()) {
        fhirServer.registerInterceptor(binaryStorageInterceptor);
    }

    if (appProperties.getValidation().getRequests_enabled()) {
        RequestValidatingInterceptor interceptor = new RequestValidatingInterceptor();
        interceptor.setFailOnSeverity(ResultSeverityEnum.ERROR);
        fhirServer.registerInterceptor(interceptor);
    }

    repositoryValidatingInterceptor.ifPresent(fhirServer::registerInterceptor);

    return fhirServer;
}
```

Standard interceptors used:

| Interceptor | Purpose | Default State |
|-------------|---------|---------------|
| `ResponseHighlighterInterceptor` | HTML output for browsers | Always on |
| `LoggingInterceptor` | Request/response logging | Always on |
| `CorsInterceptor` | CORS headers | Enabled when `cors` configured |
| `RequestValidatingInterceptor` | Request validation | Off by default |
| `ResponseValidatingInterceptor` | Response validation | Off by default |
| `FhirPathFilterInterceptor` | FHIRPath result filtering | Off by default |
| `SubscriptionDebugLogInterceptor` | Subscription debugging | When subscriptions enabled |
| `CascadingDeleteInterceptor` | Cascade delete handling | Off by default |
| `BinaryStorageInterceptor` | Binary attachment handling | Off by default |
| `OpenApiInterceptor` | Swagger UI | Enabled by default |
| `RepositoryValidatingInterceptor` | Profile-based validation | Off by default |
| `UserRequestRetryVersionConflictsInterceptor` | Optimistic locking retry | Off by default |

## Consequences

### Positive

- **Rich Hook Points**: HAPI provides hooks for every stage of request processing (SERVER_INCOMING_REQUEST_PRE_PROCESSED, STORAGE_PRESTORAGE_RESOURCE_CREATED, etc.)
- **FHIR-Aware Context**: Interceptors receive parsed FHIR resources, operation details, and request context
- **Composable Pipeline**: Multiple interceptors can process the same request in sequence
- **Standard Library**: Many common needs are met by HAPI's built-in interceptors
- **Easy Extension**: Custom interceptors can be added via `custom-interceptor-classes` configuration
- **Testable**: Interceptors can be unit tested independently of the server

### Negative

- **Ordering Complexity**: Interceptor execution order matters and may require careful configuration
- **Performance Overhead**: Each interceptor adds processing time; many interceptors compound latency
- **Debugging Difficulty**: Tracing issues across multiple interceptors can be challenging
- **Coupling to HAPI**: Interceptors are specific to HAPI FHIR; not reusable in other frameworks
- **Hook Point Knowledge**: Effective use requires understanding HAPI's extensive hook point system

### Neutral

- Interceptors are registered at startup; dynamic registration is possible but not typically used
- Some interceptors are Spring beans (injected via constructor) while others are instantiated directly
- The `Optional<?>` pattern is used for conditionally-available interceptor beans
