<!--
Sync Impact Report
==================
Version change: 0.0.0 → 1.0.0 (MAJOR - initial constitution)
Modified principles: N/A (initial version)
Added sections:
  - Core Principles (5 principles)
  - Production Requirements
  - Development Workflow
  - Governance
Removed sections: N/A
Templates requiring updates:
  ✅ .specify/templates/plan-template.md (no changes needed - Constitution Check section is generic)
  ✅ .specify/templates/spec-template.md (no changes needed - requirements structure compatible)
  ✅ .specify/templates/tasks-template.md (no changes needed - phase structure compatible)
Follow-up TODOs: None
-->

# HAPI FHIR JPA Server Starter Constitution

## Core Principles

### I. FHIR Standards Compliance

All implementations MUST adhere to HL7 FHIR specifications for the configured version (DSTU2, DSTU3, R4, R4B, or R5). Resources, operations, and interactions MUST comply with the FHIR standard to ensure interoperability with other healthcare systems.

**Rationale**: Healthcare interoperability depends on strict standards adherence. Non-compliant implementations create integration failures and potential patient safety risks.

### II. Configuration-Driven Extensibility

Features MUST be toggleable via `application.yaml` configuration without code changes. Custom functionality MUST integrate through HAPI FHIR's interceptor and provider mechanisms rather than modifying core server behavior.

**Rules**:
- New features require corresponding configuration properties
- Custom interceptors register via `hapi.fhir.custom-interceptor-classes`
- Custom providers register via `hapi.fhir.custom-provider-classes`
- Default configuration values MUST be safe and production-appropriate

**Rationale**: The starter project serves diverse deployment scenarios. Configuration-driven behavior enables customization without forking.

### III. Test-First Development

All new functionality MUST include appropriate tests before or alongside implementation. Unit tests (`*Test.java`) for isolated logic; integration tests (`*IT.java`) for FHIR operations and database interactions.

**Rules**:
- Unit tests use Surefire (`mvn test`)
- Integration tests use Failsafe (`mvn verify`)
- Integration tests default to H2; document any database-specific requirements
- Test fixtures organize by FHIR version under `src/test/resources/`

**Rationale**: Healthcare software requires high reliability. Tests ensure FHIR compliance and prevent regressions in critical patient data handling.

### IV. Code Quality Gates

All code MUST pass Spotless formatting checks before merge. Constructor injection with `final` fields is REQUIRED for Spring beans. Class naming MUST use descriptive suffixes (`*Provider`, `*Service`, `*Config`, `*Interceptor`).

**Rules**:
- Run `mvn spotless:apply` before committing
- Java 17+ language features permitted
- YAML configuration uses kebab-case keys
- Package structure follows `ca.uhn.fhir.jpa.starter.*` convention

**Rationale**: Consistent code style reduces cognitive load and merge conflicts in a community-maintained project.

### V. Security-First Mindset

Security considerations MUST be documented for all new features. The starter provides NO default security implementation by design - implementers MUST add authentication, authorization, and audit logging appropriate to their deployment.

**Non-negotiable requirements**:
- Never commit credentials, API keys, or secrets
- Document security implications in PRs
- Reference HAPI FHIR security documentation for implementation guidance
- Consider BALP interceptor for audit logging in production

**Rationale**: Healthcare data (PHI/PII) requires rigorous protection. Explicit security awareness prevents accidental exposure.

## Production Requirements

Production deployments of this starter MUST address the following gaps not provided by default:

1. **Authentication/Authorization**: Implement via HAPI FHIR security interceptors
2. **Audit Logging**: Configure BALP interceptor for enterprise audit trail
3. **Subscription Topic Cache**: Replace in-memory cache with distributed implementation for clustered deployments
4. **Message Broker**: Replace in-memory broker with external broker (RabbitMQ, Kafka) for clustered deployments
5. **Database**: Replace H2 with PostgreSQL or MS SQL Server for persistence

These are documented warnings, not implementation requirements for the starter itself.

## Development Workflow

### Build Verification

Before submitting changes:
1. `mvn clean install` - Full build with tests
2. `mvn spotless:check` - Verify formatting
3. `mvn verify` - Run integration tests

### Branch Strategy

- Feature branches: `feature/descriptive-name`
- Bug fixes: `fix/issue-description`
- PRs target `master` branch

### Code Review Checklist

- [ ] FHIR compliance verified for version-specific changes
- [ ] Configuration properties documented
- [ ] Tests added/updated
- [ ] Spotless formatting passes
- [ ] Security implications considered
- [ ] Breaking changes documented

## Governance

This constitution governs development practices for the HAPI FHIR JPA Server Starter project. All contributors and AI assistants MUST verify compliance.

**Amendment Process**:
1. Propose changes via PR modifying this document
2. Document rationale for additions/modifications
3. Update dependent templates if principle changes affect them
4. Increment version according to semantic versioning

**Version Policy**:
- MAJOR: Principle removal or fundamental redefinition
- MINOR: New principle added or section expansion
- PATCH: Clarification, typo fixes, non-semantic refinements

**Compliance Review**:
- PRs MUST reference Constitution Check results
- Violations require documented justification in Complexity Tracking section of implementation plans

**Version**: 1.0.0 | **Ratified**: 2026-01-31 | **Last Amended**: 2026-01-31
