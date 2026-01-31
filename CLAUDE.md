# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

HAPI FHIR JPA Server Starter - a production-ready FHIR server implementation built on HAPI FHIR. Supports FHIR versions DSTU2, DSTU3, R4, R4B, and R5 (configured via `hapi.fhir.fhir_version` in application.yaml).

## Build and Run Commands

```bash
# Build with tests
mvn clean install

# Build without tests
mvn clean install -DskipTests

# Run locally (Spring Boot, port 8080)
mvn spring-boot:run -Pboot

# Run with Jetty
mvn -Pjetty spring-boot:run

# Build bootable WAR and run
mvn clean package spring-boot:repackage -Pboot && java -jar target/ROOT.war

# Docker
docker-compose up -d --build
```

## Testing

```bash
# Unit tests only (Surefire)
mvn test

# Unit + integration tests (Surefire + Failsafe)
mvn verify

# Run a single test class
mvn test -Dtest=CustomInterceptorTest

# Run a single integration test
mvn verify -Dit.test=ExampleServerR4IT

# Skip tests during build
mvn install -DskipTests
```

**Test naming conventions:**
- Unit tests: `*Test.java`
- Integration tests: `*IT.java` (detected by Failsafe plugin)

**Note:** Integration tests use H2 by default. If you switch to PostgreSQL, either skip tests (`-DskipTests`) or update test configuration.

## Code Formatting

```bash
# Check formatting
mvn spotless:check

# Auto-fix formatting
mvn spotless:apply
```

Spotless runs automatically in CI on PRs.

## Architecture

### Entry Point
`ca.uhn.fhir.jpa.starter.Application` - Spring Boot application serving FHIR REST API at `/fhir/*`

### Key Packages (`ca.uhn.fhir.jpa.starter`)
- `common/` - FHIR version-specific configs (`FhirServerConfigR4.java`, `FhirServerConfigR5.java`, etc.)
- `annotations/` - Conditional beans for FHIR version selection (`OnR4Condition`, `OnR5Condition`)
- `cr/` - Clinical Reasoning / CQL support
- `cdshooks/` - CDS Hooks implementation
- `mdm/` - Master Data Management configuration
- `mcp/` - Model Context Protocol for AI integration
- `elastic/` - Elasticsearch support
- `ig/` - Implementation Guide operations

### Configuration
Primary config: `src/main/resources/application.yaml`

Key configuration sections:
- `hapi.fhir.fhir_version` - FHIR version (R4, R5, etc.)
- `hapi.fhir.cr.enabled` - Clinical Reasoning
- `hapi.fhir.cdshooks.enabled` - CDS Hooks
- `hapi.fhir.mdm_enabled` - Master Data Management
- `spring.datasource.*` - Database (H2 default, supports PostgreSQL, MS SQL Server)
- `spring.ai.mcp.server.enabled` - MCP endpoint for AI assistants

### Database Support
- **H2** (default, in-memory)
- **PostgreSQL** - use dialect `ca.uhn.fhir.jpa.model.dialect.HapiFhirPostgresDialect`
- **MS SQL Server** - auto-detected dialect

### Build Profiles
- `boot` (default) - Spring Boot embedded Tomcat
- `jetty` - Jetty server
- `cloudsql-postgres` - Google Cloud SQL PostgreSQL

## Coding Conventions

- Java 17+
- Constructor injection with `final` fields
- Class naming: descriptive suffixes (`*Provider`, `*Service`, `*Config`)
- Package structure mirrors under `ca.uhn.fhir.jpa.starter`
- YAML config: kebab-case keys
- Test fixtures: `src/test/resources/` organized by FHIR version (r4/, dstu3/)

## Production Considerations

This starter lacks several enterprise features that must be added for production:
1. Security implementation (see HAPI FHIR security docs)
2. Enterprise audit logging (BALP interceptor)
3. Shared subscription topic cache (default is in-memory, not cluster-safe)
4. Shared message broker (default is in-memory, not cluster-safe)

## MCP Integration

MCP endpoint at `/mcp/messages` for AI assistant integration. Configure in Claude/Cursor:
```json
{
  "mcpServers": {
    "hapi": {
      "url": "http://localhost:8080/mcp/messages"
    }
  }
}
```
