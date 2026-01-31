# 9. Distroless Container with Non-Root Execution

Date: 2026-01-31

## Status

Accepted

## Context

The HAPI FHIR JPA Server needs a container image for deployment to Kubernetes and container orchestration platforms. The container strategy must address:

1. **Security** - Minimize attack surface, avoid running as root
2. **Size** - Smaller images mean faster pulls and reduced storage
3. **Compliance** - Meet security scanning and audit requirements
4. **Flexibility** - Support both development debugging and production hardening
5. **Compatibility** - Work with common orchestration platforms

Container base image options considered:

- **Standard Linux (Ubuntu, Debian)** - Full OS, large attack surface, debugging tools available
- **Alpine Linux** - Small, musl libc, potential compatibility issues with Java
- **Distroless** - Minimal runtime, no shell, hardened by design
- **Scratch** - Empty base, requires static binaries
- **Eclipse Temurin** - Official Java images, well-maintained

## Decision

We will use Google's distroless Java 21 image as the default container base, running as a non-root user (UID 65532). A Tomcat-based alternative is provided for traditional deployment scenarios.

Multi-stage Dockerfile:

```dockerfile
# Build stage - full Maven environment
FROM docker.io/library/maven:3.9.12-eclipse-temurin-17 AS build-hapi
WORKDIR /tmp/hapi-fhir-jpaserver-starter
# ... build steps ...

# Repackage for Spring Boot
FROM build-hapi AS build-distroless
RUN mvn package -DskipTests spring-boot:repackage -Pboot
RUN mkdir /app && cp /tmp/hapi-fhir-jpaserver-starter/target/ROOT.war /app/main.war

# Production image - distroless
FROM gcr.io/distroless/java21-debian13:nonroot AS default
USER 65532:65532
WORKDIR /app

COPY --chown=nonroot:nonroot --from=build-distroless /app /app
COPY --chown=nonroot:nonroot --from=build-hapi /tmp/hapi-fhir-jpaserver-starter/opentelemetry-javaagent.jar /app

ENTRYPOINT ["java", "--class-path", "/app/main.war", "-Dloader.path=main.war!/WEB-INF/classes/,main.war!/WEB-INF/,/app/extra-classes", "org.springframework.boot.loader.PropertiesLauncher"]
```

Alternative Tomcat target:

```dockerfile
FROM docker.io/library/tomcat:10-jre21-temurin-noble AS tomcat
USER root
RUN rm -rf /usr/local/tomcat/webapps/ROOT && \
    mkdir -p /usr/local/tomcat/data/hapi/lucenefiles && \
    chown -R 65532:65532 /usr/local/tomcat/data/hapi/lucenefiles
USER 65532
COPY --chown=65532:65532 --from=build-hapi ... /usr/local/tomcat/webapps/ROOT.war
```

## Consequences

### Positive

- **Minimal Attack Surface**: Distroless contains only the JRE and required libraries; no shell, package manager, or utilities
- **Non-Root Execution**: UID 65532 (nonroot) prevents privilege escalation; Kubernetes can verify with `runAsNonRoot: true`
- **Smaller Image Size**: Distroless images are significantly smaller than standard Linux bases (approximately 200MB vs 500MB+)
- **Security Scanning**: Fewer packages mean fewer vulnerabilities to scan and remediate
- **Compliance Friendly**: Meets many security benchmark requirements (CIS, NIST) out of the box
- **Kubernetes Native**: Numeric UID allows Kubernetes to easily detect non-root execution
- **OpenTelemetry Ready**: Java agent included for observability integration

### Negative

- **No Shell Access**: Cannot `exec` into container for debugging; troubleshooting is more difficult
- **No Package Manager**: Cannot install additional tools at runtime for diagnostics
- **Java-Only**: Distroless Java image restricts to Java workloads; cannot run helper scripts
- **Build Complexity**: Multi-stage build with separate build and runtime stages adds Dockerfile complexity
- **Tomcat Alternative**: Must explicitly target Tomcat image if traditional deployment is needed

### Neutral

- The `nonroot` tag uses a consistent UID (65532) across distroless images
- Java 21 selected for LTS support and modern features
- OpenTelemetry agent included but not activated by default (requires JVM flags)
- The `extra-classes` loader path enables runtime class additions without rebuilding
- Tomcat image uses official Tomcat base rather than distroless for compatibility with traditional deployment patterns
