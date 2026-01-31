# 3. Single-Module Maven Structure with WAR Packaging

Date: 2026-01-31

## Status

Accepted

## Context

The HAPI FHIR JPA Server Starter needs a build structure that supports:

1. **Multiple Deployment Targets** - Embedded servers (Tomcat, Jetty) and traditional application servers
2. **Simple Getting Started Experience** - Minimal configuration for new users
3. **Customization** - Easy addition of custom interceptors, providers, and configuration
4. **Container Deployment** - Docker image creation with minimal complexity
5. **WAR Overlays** - Integration with HAPI FHIR test page overlay

Several project structures were considered:

- **Single-module WAR** - Simple structure, single artifact output
- **Multi-module with core/web separation** - More complex but separates concerns
- **Multi-module with database-specific modules** - Allows database-specific packaging
- **Gradle-based build** - Alternative to Maven with better dependency management

## Decision

We will use a single-module Maven project with WAR packaging (`<packaging>war</packaging>`) that produces a deployable artifact named `ROOT.war`.

Key implementation:

```xml
<artifactId>hapi-fhir-jpaserver-starter</artifactId>
<packaging>war</packaging>

<build>
    <finalName>ROOT</finalName>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                    <configuration>
                        <mainClass>ca.uhn.fhir.jpa.starter.Application</mainClass>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-war-plugin</artifactId>
            <configuration>
                <overlays>
                    <overlay>
                        <groupId>ca.uhn.hapi.fhir</groupId>
                        <artifactId>hapi-fhir-testpage-overlay</artifactId>
                    </overlay>
                </overlays>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Build profiles provide deployment flexibility:
- `boot` (default) - Embedded Tomcat via Spring Boot Starter Web
- `jetty` - Embedded Jetty server
- `cloudsql-postgres` - Google Cloud SQL PostgreSQL support

## Consequences

### Positive

- **Simplicity**: Single `pom.xml` makes the project easy to understand and fork
- **Dual Deployment Mode**: Spring Boot repackaging creates an executable WAR that works both as standalone (`java -jar ROOT.war`) and deployed to application servers
- **WAR Overlay Support**: Maven WAR plugin enables HAPI FHIR test page overlay integration, adding a web UI for testing
- **Naming Convention**: `ROOT.war` naming allows default context path deployment on Tomcat without additional configuration
- **Fork-Friendly**: Users can clone and customize without understanding multi-module Maven structures
- **Profile-Based Flexibility**: Build profiles allow switching embedded servers without code changes

### Negative

- **Monolithic Artifact**: All dependencies bundled together; no separation of core logic from web layer
- **Limited Reusability**: Cannot publish a library JAR for use as a dependency in other projects (though `attachClasses` partially addresses this)
- **Build Complexity**: Profiles and overlays add Maven configuration complexity
- **Artifact Size**: Single WAR includes all optional features even if not used; no modular packaging

### Neutral

- The `<finalName>ROOT</finalName>` convention is Tomcat-specific but works with other containers
- Spring Boot repackaging with `CLASSIC` loader implementation addresses compatibility issues with Hibernate Search
- Resource filtering with `@` delimiter enables Maven property substitution in configuration files
