# 5. Configuration-Driven Feature Toggles

Date: 2026-01-31

## Status

Accepted

## Context

The HAPI FHIR JPA Server includes numerous optional features that organizations may or may not need:

- Clinical Reasoning (CQL) operations
- CDS Hooks integration
- Master Data Management (MDM)
- Bulk data export/import
- GraphQL support
- Binary storage
- Subscriptions (REST hook, WebSocket, email)
- OpenAPI/Swagger documentation
- Request/response validation
- Full-text search (Lucene/Elasticsearch)

These features have different:
- **Resource Requirements** - Some are memory/CPU intensive
- **Dependencies** - External services or additional configuration
- **Security Implications** - Some expose sensitive operations
- **Use Cases** - Not all deployments need all features

Approaches considered:

- **Compile-Time Feature Flags** - Exclude features via Maven profiles; requires separate builds
- **Runtime Configuration** - Enable/disable via properties; single build, flexible deployment
- **Module-Based Packaging** - Separate JARs per feature; complex dependency management
- **Feature Annotations** - Custom annotations for feature gating

## Decision

We will use Spring Boot's `@ConfigurationProperties` with a comprehensive `AppProperties` class that maps the `hapi.fhir.*` configuration namespace. Features are enabled/disabled via boolean properties with sensible defaults (typically disabled for optional features).

Implementation:

**AppProperties Class**:

```java
@EnableConfigurationProperties
@ConfigurationProperties(prefix = "hapi.fhir")
@Configuration
public class AppProperties {
    private Boolean cr_enabled = false;
    private Boolean mdm_enabled = false;
    private Boolean bulk_export_enabled = false;
    private Boolean graphql_enabled = false;
    private Boolean openapi_enabled = false;
    private Boolean binary_storage_enabled = false;
    // ... 100+ configurable properties
}
```

**Conditional Bean Registration**:

```java
@Bean
@ConditionalOnProperty(prefix = "hapi.fhir", name = "binary_storage_mode", havingValue = "FILESYSTEM")
public FilesystemBinaryStorageSvcImpl filesystemBinaryStorageSvc(AppProperties appProperties) {
    return new FilesystemBinaryStorageSvcImpl(appProperties.getBinary_storage_filesystem_base_directory());
}
```

**Feature Registration in RestfulServer**:

```java
if (appProperties.getGraphql_enabled()) {
    if (fhirSystemDao.getContext().getVersion().getVersion().isEqualOrNewerThan(FhirVersionEnum.DSTU3)) {
        fhirServer.registerProvider(graphQLProvider.get());
    }
}

if (appProperties.getBulk_export_enabled()) {
    fhirServer.registerProvider(bulkDataExportProvider);
}
```

**Configuration in application.yaml**:

```yaml
hapi:
  fhir:
    cr:
      enabled: false
    mdm_enabled: false
    bulk_export_enabled: true
    graphql_enabled: true
    openapi_enabled: true
```

## Consequences

### Positive

- **Single Build, Multiple Configurations**: One artifact supports all deployment scenarios via configuration
- **Safe Defaults**: Features default to disabled, requiring explicit opt-in for security-sensitive operations
- **Runtime Flexibility**: Features can be toggled without rebuilding; useful for staging vs. production
- **Type-Safe Configuration**: `AppProperties` provides IDE autocomplete and compile-time validation of property names
- **Centralized Configuration**: All HAPI settings in one class with consistent naming conventions
- **Environment Override**: Properties can be overridden via environment variables (`HAPI_FHIR_MDM_ENABLED=true`)
- **Documentation**: Property names serve as self-documenting feature catalog

### Negative

- **Large Configuration Class**: `AppProperties` has 100+ fields, making it complex to maintain
- **Naming Inconsistency**: Mix of underscore (`cr_enabled`) and camelCase (`customInterceptorClasses`) naming
- **Hidden Dependencies**: Some features require additional configuration beyond the enable flag (e.g., MDM needs `mdm_rules_json_location`)
- **Startup Validation**: Invalid configurations may not fail fast; errors may occur at runtime
- **Documentation Gap**: Not all property combinations and their effects are documented

### Neutral

- Properties use snake_case to match YAML convention, though this differs from Java camelCase norms
- Nested classes (`Subscription`, `Cors`, `Partitioning`) organize related properties
- Default values are set in field declarations rather than YAML to ensure consistency
