# Architecture Decision Records

This directory contains Architecture Decision Records (ADRs) documenting the key architectural decisions made in the HAPI FHIR JPA Server Starter project.

## What is an ADR?

An Architecture Decision Record (ADR) is a document that captures an important architectural decision made along with its context and consequences. ADRs help maintain institutional knowledge and provide rationale for why the system is built the way it is.

## ADR Index

| ADR | Title | Status |
|-----|-------|--------|
| [0001](0001-record-architecture-decisions.md) | Record Architecture Decisions | Accepted |
| [0002](0002-spring-boot-framework.md) | Spring Boot as Core Framework | Accepted |
| [0003](0003-single-module-maven-war.md) | Single-Module Maven Structure with WAR Packaging | Accepted |
| [0004](0004-conditional-bean-loading.md) | Conditional Bean Loading for Multi-FHIR Version Support | Accepted |
| [0005](0005-configuration-driven-features.md) | Configuration-Driven Feature Toggles | Accepted |
| [0006](0006-h2-default-multi-database.md) | H2 as Default Database with Multi-Database Support | Accepted |
| [0007](0007-interceptor-pattern.md) | HAPI Interceptor Pattern for Cross-Cutting Concerns | Accepted |
| [0008](0008-custom-extension-points.md) | Custom Bean/Interceptor/Provider Extension Points | Accepted |
| [0009](0009-distroless-container.md) | Distroless Container with Non-Root Execution | Accepted |
| [0010](0010-hierarchical-yaml-configuration.md) | Hierarchical YAML Configuration with Environment Override | Accepted |
| [0011](0011-test-strategy-surefire-failsafe.md) | Test Strategy with Surefire/Failsafe Separation | Accepted |

## ADR Template

New ADRs should follow this template:

```markdown
# [Number]. [Title]

Date: YYYY-MM-DD

## Status

[Proposed | Accepted | Deprecated | Superseded by ADR-XXX]

## Context

[Describe the issue that motivates this decision or change]

## Decision

[Describe the change or decision being made]

## Consequences

[Describe the resulting context, after applying the decision]
```

## Contributing

When making significant architectural changes, please create a new ADR documenting the decision. ADRs are numbered sequentially and should never be deleted - instead, mark them as deprecated or superseded.

For a lightweight ADR toolset, see Nat Pryce's [adr-tools](https://github.com/npryce/adr-tools).