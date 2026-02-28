# PRD: Security Hardening

## Introduction

Address critical security vulnerabilities discovered during the system audit to protect the Depfender platform from container escape attacks, SSRF exploits, prompt injection, and unauthorized access. This PRD covers comprehensive hardening across authentication, container security, input validation, and endpoint protection.

## Goals

- Eliminate container escape risk by removing Docker socket mount
- Prevent SSRF attacks in package fetching by validating tarball URLs
- Add authentication to all sensitive endpoints (`/metrics`, `/api/docs`)
- Sanitize all user/package input before LLM prompts to prevent prompt injection
- Implement API key rotation mechanism for `INTERNAL_API_SECRET`
- Add security headers to all HTTP responses
- Ensure no sensitive data leakage in error messages

## User Stories

### US-001: Remove Docker Socket Mount from Worker
**Description:** As a security engineer, I want the worker container to communicate with Docker without socket mounting so that a compromised container cannot escape to the host.

**Acceptance Criteria:**
- [ ] Worker communicates with Docker daemon over TCP (port 2376) instead of socket mount
- [ ] Docker daemon configured with TLS authentication for TCP connections
- [ ] docker-compose.yml no longer contains `/var/run/docker.sock` mount
- [ ] Analysis containers still spawn correctly
- [ ] Unit tests pass for container spawning
- [ ] Integration test: spawn analysis container and verify isolation

### US-002: Validate Tarball URLs Against Allowlist
**Description:** As a security engineer, I want tarball download URLs validated against an allowlist so that attackers cannot redirect downloads to internal services or malicious endpoints.

**Acceptance Criteria:**
- [ ] Create URL allowlist: `registry.npmjs.org`, `github.com`, `gitlab.com`, `bitbucket.org`
- [ ] `packageResolver.ts` validates URL hostname before fetch
- [ ] Reject URLs with IP addresses (prevent SSRF to internal networks)
- [ ] Reject URLs with non-standard ports
- [ ] Reject URLs with authentication credentials embedded
- [ ] Log rejected URLs with reason for security monitoring
- [ ] Unit tests for URL validation (valid hosts, IPs, ports, credentials)
- [ ] Integration test: attempt fetch from blocked host returns error

### US-003: Add Authentication to /metrics Endpoint
**Description:** As a security engineer, I want the `/metrics` endpoint protected so that attackers cannot gather intelligence about system internals.

**Acceptance Criteria:**
- [ ] `/metrics` requires `x-internal-api-secret` header or separate metrics token
- [ ] Unauthenticated requests return 401
- [ ] Prometheus scraper configured with authentication header
- [ ] Unit test: unauthenticated request returns 401
- [ ] Integration test: authenticated request returns metrics

### US-004: Add Authentication to /api/docs Endpoint
**Description:** As a security engineer, I want the `/api/docs` endpoint protected in production so that API documentation is not publicly exposed.

**Acceptance Criteria:**
- [ ] `/api/docs` requires authentication in production environment
- [ ] `/api/docs` remains open in development environment for convenience
- [ ] Environment variable `DOCS_PUBLIC=true|false` controls behavior
- [ ] Unauthenticated production requests return 401
- [ ] Unit test: production mode blocks unauthenticated access

### US-005: Sanitize Package Metadata in LLM Prompts
**Description:** As a security engineer, I want package metadata sanitized before injection into LLM prompts so that malicious packages cannot manipulate the security analysis.

**Acceptance Criteria:**
- [ ] Create `sanitizeForPrompt()` function that escapes/removes injection patterns
- [ ] Remove markdown formatting, code blocks, system prompt patterns from metadata
- [ ] Truncate excessively long fields (name: 214 chars, description: 1000 chars)
- [ ] Apply sanitization to: package name, version, description, author, keywords
- [ ] Log original vs sanitized values when changes made
- [ ] Unit tests for injection patterns: `</system>`, `Human:`, code blocks, etc.
- [ ] Integration test: scan package with malicious description, verify sanitized

### US-006: Add Security Headers Middleware
**Description:** As a security engineer, I want security headers on all responses so that browser-based attacks are mitigated.

**Acceptance Criteria:**
- [ ] Add `helmet` middleware with secure defaults
- [ ] Headers include: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security` (production only)
- [ ] CSP header configured for API-only server (no inline scripts needed)
- [ ] Unit test: verify headers present on response

### US-007: Prevent Sensitive Data in Error Messages
**Description:** As a security engineer, I want error messages to not leak sensitive information so that attackers cannot gather system details from errors.

**Acceptance Criteria:**
- [ ] Create error sanitization middleware
- [ ] Production errors return generic message with error ID only
- [ ] Stack traces only included in development mode
- [ ] Database errors do not expose table/column names
- [ ] File paths are not exposed in errors
- [ ] Log full error details server-side with error ID for correlation
- [ ] Unit tests: trigger each error type, verify sanitized response

### US-008: Implement API Secret Rotation Mechanism
**Description:** As a security engineer, I want to rotate the API secret without downtime so that compromised secrets can be quickly invalidated.

**Acceptance Criteria:**
- [ ] Support comma-separated list of valid secrets: `INTERNAL_API_SECRET=new,old`
- [ ] Authentication accepts any valid secret in the list
- [ ] Add `GET /health/secret-check` endpoint to verify which secret is in use (authenticated)
- [ ] Document rotation procedure in README
- [ ] Unit test: both old and new secrets work during rotation window

## Functional Requirements

- FR-1: The system must communicate with Docker daemon over TLS-authenticated TCP, not socket mount
- FR-2: The system must validate all tarball URLs against hostname allowlist before fetching
- FR-3: The system must reject tarball URLs containing IP addresses, non-standard ports, or credentials
- FR-4: The system must require authentication for `/metrics` endpoint
- FR-5: The system must require authentication for `/api/docs` in production environments
- FR-6: The system must sanitize all package metadata before including in LLM prompts
- FR-7: The system must truncate package name to 214 chars and description to 1000 chars
- FR-8: The system must include security headers (X-Content-Type-Options, X-Frame-Options, HSTS) on all responses
- FR-9: The system must return generic error messages in production without stack traces or internal details
- FR-10: The system must support multiple valid API secrets for zero-downtime rotation

## Non-Goals

- OAuth2/JWT authentication system (future consideration)
- Per-user API keys and rate limiting
- Web Application Firewall (WAF) integration
- Intrusion detection system integration
- Secrets management service integration (Vault, AWS Secrets Manager)

## Technical Considerations

- **Docker TCP:** Requires Docker daemon configured with `--host tcp://0.0.0.0:2376 --tlsverify`
- **TLS Certs:** Need to generate and mount TLS certificates for Docker client authentication
- **Helmet:** Use `helmet` npm package for security headers
- **Sanitization:** Use allowlist approach (keep safe chars) rather than blocklist (remove dangerous chars)
- **Backward Compatibility:** Existing `INTERNAL_API_SECRET` single value still works

## Success Metrics

- Zero unauthenticated access to `/metrics` or `/api/docs` in production logs
- Zero SSRF attempts succeed (all blocked and logged)
- No stack traces or internal paths visible in production error responses
- Security scan (e.g., OWASP ZAP) passes with no high/critical findings

## Open Questions

- Should we implement IP allowlisting for `/metrics` as additional protection?
- Should we add request signing for frontend-to-backend calls?
- Do we need audit logging for authentication failures?
