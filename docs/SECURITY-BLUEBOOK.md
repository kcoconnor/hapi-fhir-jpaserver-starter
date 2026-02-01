# Security Blue Book v1.0
## HAPI FHIR JPA Server Starter

**Document Version:** 1.0
**Last Updated:** 2026-02-01
**Status:** Development Environment - Pre-Production

---

## Normative Language

- **MUST**: Required for compliance and release to production
- **SHOULD**: Recommended best practice; deviation requires documented justification
- **CAN**: Optional or context-dependent

---

## Scope

| Attribute | Value |
|-----------|-------|
| **Product** | HAPI FHIR JPA Server Starter |
| **FHIR Version** | R4 (configurable: DSTU2, DSTU3, R4, R4B, R5) |
| **Environment** | Development (pre-production) |
| **Data Classes** | PHI, PII, Financial (billing/claims) |
| **Regulatory Framework** | HIPAA (future requirement) |

### Out of Scope
- Physical security controls
- Network perimeter security (firewall rules)
- Employee background checks
- Business associate agreement (BAA) management
- Disaster recovery site selection

---

## Threat Model (MVP)

### Assumptions
1. The server operates within a trusted network perimeter in development
2. All client applications are known and registered (no anonymous access in production)
3. Database encryption is handled by the underlying database platform
4. TLS termination occurs at a load balancer or reverse proxy in production
5. Operators have appropriate background checks and training

### Primary Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Unauthorized PHI access | Critical | Medium | SMART on FHIR authorization, audit logging |
| SQL injection | Critical | Low | JPA/Hibernate parameterized queries, input validation |
| Session hijacking | High | Medium | Short-lived tokens, secure cookies, HTTPS only |
| Data exfiltration via bulk export | Critical | Medium | Authorization scopes, rate limiting, audit trail |
| MCP endpoint abuse | High | Medium | Disable in production or add authentication |
| Credential exposure in logs | High | Medium | Log scrubbing, secret detection |

### Out-of-Scope Threats
- Nation-state adversaries with physical access
- Insider threats with database administrator access
- Zero-day vulnerabilities in JVM/OS
- DDoS attacks (handled at infrastructure layer)

---

## Data Classification and Handling

### PHI (Protected Health Information)
- **Examples**: Patient resources, Observations, Conditions, MedicationRequests, DiagnosticReports
- **Storage**: MUST be encrypted at rest in production
- **Access**: MUST require authenticated session with appropriate FHIR scopes
- **Logging**: MUST log all access events; MUST NOT log PHI content in logs
- **Transmission**: MUST use TLS 1.2+ in production
- **Retention**: MUST retain for minimum 6 years per HIPAA

### PII (Personally Identifiable Information)
- **Examples**: Patient demographics (name, DOB, address, SSN), Practitioner contact info
- **Storage**: Same as PHI
- **Access**: MUST be scoped to patient context where applicable
- **Display**: SHOULD mask SSN, display only last 4 digits when shown

### Financial Data
- **Examples**: Coverage, Claim, ExplanationOfBenefit resources
- **Storage**: Same as PHI
- **Access**: MUST require explicit financial scopes
- **Audit**: MUST log all financial resource access

### Tokens/Credentials
- **Examples**: OAuth tokens, API keys, database passwords
- **Storage**: MUST NOT store in source code or application.yaml committed to git
- **Runtime**: MUST use environment variables or secrets manager
- **Rotation**: SHOULD rotate at least annually; MUST rotate on suspected compromise
- **Logging**: MUST NEVER appear in logs

---

## Trust Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│                     UNTRUSTED ZONE                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Patient App │  │ Provider   │  │ Third-party │         │
│  │ (SMART)     │  │ Portal     │  │ Systems     │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
└─────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │
          ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                   NETWORK BOUNDARY                          │
│            (Load Balancer / API Gateway)                   │
│            - TLS termination                               │
│            - Rate limiting                                 │
│            - IP allowlisting (if applicable)              │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                   APPLICATION BOUNDARY                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              HAPI FHIR Server (:8080)               │   │
│  │  - /fhir/*  : FHIR REST API                        │   │
│  │  - /mcp/*   : MCP endpoint (TODO: secure/disable)  │   │
│  │  - /actuator/health : Health check only            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                     DATA BOUNDARY                           │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │   PostgreSQL    │  │  Elasticsearch  │                  │
│  │   (encrypted)   │  │   (optional)    │                  │
│  └─────────────────┘  └─────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

### Controls by Boundary

| Boundary | Control | Status |
|----------|---------|--------|
| Network | TLS 1.2+ required | TODO - configure in production |
| Network | Rate limiting | TODO - implement at gateway |
| Application | SMART on FHIR auth | TODO - implement interceptor |
| Application | CORS restricted | PARTIAL - currently allows `*` |
| Application | MCP endpoint secured | TODO - currently unauthenticated |
| Data | Encryption at rest | TODO - configure database |
| Data | Audit logging | TODO - implement BALP |

---

## Authentication and Session Rules

### Authentication Method: SMART on FHIR
**Status**: TODO - Not yet implemented

When implemented, the following rules MUST apply:

1. **OAuth 2.0 Authorization Server**
   - MUST use an external authorization server (Keycloak, Auth0, or similar)
   - MUST support PKCE for public clients
   - MUST validate `aud` claim matches FHIR server URL

2. **Scopes**
   - MUST enforce SMART scopes (e.g., `patient/*.read`, `user/Observation.write`)
   - MUST deny requests without valid scope for the resource type
   - SHOULD use launch context scopes for EHR integration

3. **Session Lifetime**
   - Access tokens: MUST expire within 1 hour
   - Refresh tokens: SHOULD expire within 24 hours for patient-facing apps
   - Refresh tokens: CAN extend to 30 days for provider applications with re-authentication

4. **MFA Policy**
   - MUST require MFA for administrative access
   - SHOULD require MFA for provider access to PHI
   - CAN allow single-factor for patient portal with risk-based step-up

5. **Session Revocation**
   - MUST support immediate token revocation
   - MUST invalidate all sessions on password change
   - SHOULD invalidate sessions on suspicious activity detection

### Current State (Development)
- No authentication implemented
- All endpoints are open
- **MUST NOT deploy to production without implementing authentication**

---

## Token Handling Policy

### OAuth Access Tokens
- **Storage**: In-memory only on client; MUST NOT persist to disk
- **Transmission**: Authorization header only; MUST NOT pass in URL parameters
- **Validation**: MUST validate signature, expiry, issuer, and audience on every request
- **Scope**: MUST enforce principle of least privilege

### Refresh Tokens
- **Storage**: Encrypted in secure storage (keychain, encrypted DB)
- **Rotation**: MUST issue new refresh token on each use (rotation)
- **Revocation**: MUST support explicit revocation; MUST revoke on logout

### Database Credentials
- **Source**: MUST come from environment variables or secrets manager
- **Current**: `application.yaml` has `password: null` for H2 - acceptable for dev only
- **Production**: MUST use strong passwords; MUST NOT use default credentials

### API Keys (if used)
- **Issuance**: MUST track issuer, purpose, and expiry
- **Rotation**: MUST rotate every 90 days
- **Revocation**: MUST support immediate revocation

---

## Logging and Audit Policy

### MUST Log (Audit Events)
- All authentication attempts (success and failure)
- All FHIR resource access (create, read, update, delete)
- Authorization decisions (grants and denials)
- Administrative operations (config changes, user management)
- Bulk export requests and completions
- System errors and exceptions

### MUST NEVER Log
- PHI content (patient names, conditions, observations)
- PII content (SSN, full address, phone numbers)
- Authentication credentials (passwords, tokens, API keys)
- Full request/response bodies containing health data

### Redaction Rules
- Patient identifiers: Log resource ID only, not name/MRN
- Tokens: Log token type and last 4 characters only
- Search parameters: Redact patient-identifying search values
- Error messages: Sanitize before logging; no stack traces with PHI

### Audit Format
SHOULD implement FHIR AuditEvent (BALP profile) for healthcare-standard audit logging.

### Retention
- Audit logs: MUST retain for minimum 6 years
- Application logs: SHOULD retain for 90 days
- Debug logs: CAN retain for 7 days

---

## Storage and Encryption

### At Rest
| Data Store | Encryption | Status |
|------------|------------|--------|
| PostgreSQL | AES-256 via TDE | TODO - configure in production |
| H2 (dev) | None | Acceptable for synthetic data only |
| Elasticsearch | Encryption at rest | TODO - if enabled |
| Binary storage | Filesystem encryption | TODO - if binary_storage enabled |

### In Transit
- MUST use TLS 1.2 or higher for all external connections
- MUST use TLS for database connections in production
- SHOULD use mTLS for service-to-service communication

### Secret Management
- **Development**: Environment variables acceptable
- **Production**: MUST use secrets manager (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager)
- **Rotation**: MUST support rotation without downtime

---

## Retention and Deletion

### Default Retention Periods
| Data Type | Retention | Justification |
|-----------|-----------|---------------|
| PHI/PII | 6 years minimum | HIPAA requirement |
| Audit logs | 6 years minimum | HIPAA requirement |
| Financial data | 7 years | IRS requirements |
| Session data | 24 hours after expiry | No longer needed |
| Debug logs | 7 days | Troubleshooting window |

### User-Initiated Deletion
- MUST support patient right of access amendment requests
- MUST log all deletion requests and completions
- MUST NOT delete data required for legal hold
- SHOULD implement soft delete with configurable hard delete

### Deletion Verification
- MUST verify deletion from primary database
- MUST verify deletion from backups within backup rotation period
- MUST verify deletion from search indexes
- MUST generate deletion certificate for compliance

---

## Incident Response (MVP)

### 1. Detect
- Monitor application logs for authentication failures (>5 failures/minute = alert)
- Monitor for unusual data access patterns (bulk downloads, off-hours access)
- Monitor for error rate spikes (>1% error rate = investigate)
- TODO: Implement alerting infrastructure

### 2. Contain
- Revoke compromised credentials immediately
- Block suspicious IP addresses at network boundary
- Disable affected user accounts
- If breach confirmed: take system offline if necessary

### 3. Notify
- **Within 1 hour**: Notify security team lead
- **Within 24 hours**: Notify CISO and legal
- **Within 72 hours**: Notify affected individuals (if PHI breach confirmed, per HIPAA)
- **Within 60 days**: File HHS breach report (if >500 individuals affected)

### 4. Rotate
- Rotate all potentially compromised credentials
- Rotate database passwords
- Rotate API keys
- Issue new OAuth signing keys if authorization server compromised

### 5. Postmortem
- Document timeline of events
- Identify root cause
- Document remediation steps taken
- Update security controls to prevent recurrence
- File incident report for compliance records

### Emergency Contacts
| Role | Contact | Escalation |
|------|---------|------------|
| Security Lead | TODO | Immediate |
| CISO | TODO | Within 1 hour |
| Legal | TODO | Within 24 hours |
| On-call Engineer | TODO | Immediate |

---

## Security Gates (Pass/Fail)

These gates MUST pass before production deployment:

### Gate 1: Authentication Implementation
- [ ] SMART on FHIR authorization interceptor implemented
- [ ] OAuth 2.0 authorization server configured
- [ ] Token validation on all /fhir/* endpoints
- [ ] MCP endpoint secured or disabled

### Gate 2: Encryption and Secrets
- [ ] TLS 1.2+ enforced on all external endpoints
- [ ] Database encryption at rest enabled
- [ ] No credentials in source code or committed config files
- [ ] Secrets manager integration complete

### Gate 3: Audit and Compliance
- [ ] BALP audit logging implemented
- [ ] Audit log retention configured (6+ years)
- [ ] PHI redaction verified in all logs
- [ ] Access logging covers all FHIR operations

### Gate 4: CORS and Network Hardening
- [ ] CORS restricted to known origins (not `*`)
- [ ] Actuator endpoints restricted to internal network
- [ ] Rate limiting implemented
- [ ] IP allowlisting configured (if applicable)

### Gate 5: Vulnerability Assessment
- [ ] Dependency vulnerability scan clean (no critical/high)
- [ ] OWASP Top 10 assessment complete
- [ ] Penetration test completed (or scheduled)
- [ ] Security review sign-off obtained

---

## Pre-Launch Go/No-Go Checklist

### Infrastructure
- [ ] Production database provisioned with encryption at rest
- [ ] TLS certificates installed and valid
- [ ] Load balancer configured with rate limiting
- [ ] Backup and recovery tested

### Application Security
- [ ] All Security Gates (1-5) passed
- [ ] Default passwords changed
- [ ] Debug endpoints disabled (`openapi_enabled: false` if not needed)
- [ ] MCP endpoint secured or disabled (`spring.ai.mcp.server.enabled: false`)
- [ ] CORS `allowed_origin` restricted to specific domains

### Compliance
- [ ] Privacy policy published
- [ ] Terms of service published
- [ ] HIPAA compliance documentation complete
- [ ] BAA signed with all vendors handling PHI

### Operations
- [ ] Incident response runbook reviewed
- [ ] On-call rotation established
- [ ] Monitoring and alerting configured
- [ ] Log aggregation operational

### Sign-off
- [ ] Security team approval
- [ ] Compliance team approval
- [ ] Engineering lead approval
- [ ] Product owner approval

---

## Security Exceptions

If a MUST requirement cannot be met, document an exception below:

| Requirement | Exception Scope | Duration | Mitigation | Approved By |
|-------------|-----------------|----------|------------|-------------|
| (none) | | | | |

**Exception Process:**
1. Document the requirement that cannot be met
2. Describe the scope and duration of the exception
3. Document compensating controls/mitigations
4. Obtain written approval from Security Lead and CISO
5. Review exception at each security review cycle

---

## Current Configuration Concerns

Based on review of `application.yaml`, the following items require attention before production:

| Item | Current Value | Production Requirement |
|------|---------------|----------------------|
| `spring.datasource.password` | `null` | MUST use strong password |
| `hapi.fhir.cors.allowed_origin` | `*` | MUST restrict to known origins |
| `spring.ai.mcp.server.enabled` | `true` | SHOULD disable or secure |
| `hapi.fhir.openapi_enabled` | `true` | SHOULD disable in production |
| `hapi.fhir.tester` | enabled | MUST disable in production |
| Database | H2 in-memory | MUST use PostgreSQL with encryption |
| TLS | Not configured | MUST configure at gateway/LB |
| Authentication | None | MUST implement SMART on FHIR |

---

## Appendix A: SMART on FHIR Implementation Guide

When implementing SMART on FHIR authentication, refer to:
- [SMART App Launch Framework](http://hl7.org/fhir/smart-app-launch/)
- [HAPI FHIR Security Documentation](https://hapifhir.io/hapi-fhir/docs/security/introduction.html)

Key interceptors to implement:
1. `AuthorizationInterceptor` - Resource-level authorization
2. `ConsentInterceptor` - Patient consent enforcement
3. `BalpAuditCaptureInterceptor` - BALP audit logging

---

## Appendix B: Related Documents

- CLAUDE.md - Project development guidance
- TODO.md - Outstanding work items
- ADRs in /docs - Architecture decisions

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-01 | Claude Code | Initial draft |
