# Production Security, Architecture & Readiness Audit

**Purpose:** Give an AI coding agent a structured, evidence-based audit
plan to assess this repository before production deployment.

## Agent operating rules

1.  **Audit the actual repository.** Do not assume a framework,
    database, hosting provider, authentication system, or architecture.
    Discover them first.
2.  **Do not claim a control exists because a package is installed.**
    Trace the implementation and verify behavior.
3.  **Do not modify code before reporting the baseline findings.** First
    inspect, map, test safely, and produce an audit report. Then propose
    or implement fixes only when explicitly authorized.
4.  **Never run destructive tests against production.** Use a local or
    staging environment and test accounts.
5.  **Do not print, copy, or expose secret values.** Report file,
    variable name, and secret type; redact the value. If a real secret
    is exposed, recommend immediate rotation.
6.  **Do not silently skip areas.** Mark each item `PASS`, `FAIL`,
    `PARTIAL`, `NOT APPLICABLE`, or `NOT VERIFIED`, with evidence.
7.  **Every finding must include:** severity, affected
    file/route/component, evidence (file and line/function where
    possible), exploit scenario, impact, exact remediation, and a
    regression test.
8.  **Do not invent findings.** Distinguish confirmed vulnerabilities
    from risks, missing evidence, and recommendations.
9.  **Preserve existing behavior.** Any proposed code change must be
    minimal, explain compatibility impact, and include tests.
10. **Finish with a deployment decision:** `BLOCK`, `CONDITIONAL GO`, or
    `GO`, with explicit reasons.

------------------------------------------------------------------------

# Phase 0 --- Repository and Architecture Discovery

-   [ ] Identify languages, frameworks, runtime versions, package
    managers, and build tools.
-   [ ] Map frontend, backend, API routes, server actions, workers,
    scheduled jobs, and background services.
-   [ ] Identify databases, ORM/query builders, caches, queues, object
    storage, and external services.
-   [ ] Identify authentication provider, token/session mechanism,
    roles, permissions, and tenant model.
-   [ ] Identify deployment platform, reverse proxy, CDN, containers,
    CI/CD workflows, and infrastructure-as-code.
-   [ ] Inventory all exposed routes and methods, including debug,
    health, admin, webhook, and internal routes.
-   [ ] Identify data classes: credentials, personal data, payment data,
    business-sensitive data, uploaded files, and logs.
-   [ ] Produce a concise architecture/data-flow diagram and
    trust-boundary map.
-   [ ] List areas that cannot be inspected because configuration,
    credentials, services, or files are unavailable.

**Required output:** `architecture-inventory.md` with components, trust
boundaries, data flows, endpoints, and assumptions.

# Phase 1 --- Secrets and Environment Security

-   [ ] Scan tracked files, configuration, scripts, tests, build output,
    and client bundles for API keys, passwords, JWT signing keys,
    database URLs, private certificates, and service-role keys.
-   [ ] Inspect Git history for previously committed secrets. Do not
    print secret values.
-   [ ] Verify `.env`, local settings, credential files, private keys,
    and generated secret files are excluded by `.gitignore`.
-   [ ] Confirm production secrets are injected server-side through
    environment configuration or a managed secrets manager.
-   [ ] Verify secrets are not exposed through frontend environment
    prefixes such as `NEXT_PUBLIC_`, `VITE_`, or framework equivalents.
-   [ ] Confirm service-role/admin credentials are never used in
    browser/mobile code.
-   [ ] Check source maps, build artifacts, logs, error messages, and
    deployment artifacts for secrets.
-   [ ] Check secret rotation, access permissions, and separation
    between development, staging, and production.
-   [ ] If a secret was exposed, recommend revocation/rotation; deleting
    it from the current file is not sufficient.

**Expected fixes:** remove hardcoded credentials; load secrets
server-side; rotate compromised credentials; restrict secret access; add
safe example configuration such as `.env.example` containing
placeholders only.

# Phase 2 --- Authentication and Session Security

-   [ ] Verify every protected endpoint validates authentication
    server-side.
-   [ ] Verify JWT signature, allowed algorithm, issuer, audience,
    expiry, not-before, and key rotation. Reject unsigned tokens and
    algorithm confusion.
-   [ ] Do not trust client-supplied identity, role, permission, or
    tenant claims without validating them against trusted server-side
    state where required.
-   [ ] Verify password hashing uses Argon2id or an appropriately
    configured bcrypt implementation; never use plaintext, MD5, or
    unsalted fast hashes for passwords.
-   [ ] Review login, registration, email/phone verification, MFA,
    password change, account recovery, and reset flows.
-   [ ] Generate OTPs and reset tokens with a cryptographically secure
    random generator.
-   [ ] Ensure OTP/reset tokens are single-use, short-lived, securely
    stored (prefer hashes), and attempt-limited.
-   [ ] Prevent account enumeration through consistent authentication
    and recovery responses.
-   [ ] Ensure refresh-token rotation, reuse detection, revocation, and
    logout invalidation where applicable.
-   [ ] For browser sessions, use `Secure`, `HttpOnly`, and appropriate
    `SameSite` cookie settings; use appropriate expiration and rotation.
-   [ ] Do not store long-lived sensitive tokens in `localStorage` or
    other JavaScript-readable storage without a justified threat-model
    decision.
-   [ ] Verify session expiration, idle timeout, password-change
    invalidation, and privileged-session reauthentication.
-   [ ] Validate OAuth redirect URIs against an exact allowlist;
    validate `state`; use PKCE for public clients.

**Expected fixes:** centralized authentication middleware; strict token
verification; secure cookie settings; refresh-token rotation/revocation;
robust OTP/reset handling; OAuth state and redirect validation.

# Phase 3 --- Authorization, RBAC, BOLA/IDOR, and Mass Assignment

-   [ ] Inventory roles, permissions, ownership rules, and privileged
    operations.
-   [ ] Enforce authorization on the server for every protected route
    and operation.
-   [ ] Apply default-deny behavior: routes are inaccessible unless
    explicitly authorized.
-   [ ] Verify role checks occur before sensitive controller/service
    actions.
-   [ ] For every object lookup by ID, verify the authenticated user is
    authorized to access that exact object.
-   [ ] Test read, create, update, delete, export, download, and
    nested-resource operations.
-   [ ] Test horizontal escalation (user A accessing user B) and
    vertical escalation (ordinary user accessing admin functions).
-   [ ] Never rely on UUIDs or unguessable identifiers as a replacement
    for authorization.
-   [ ] Prevent mass assignment: accept explicit allowlisted fields
    rather than spreading request bodies into database updates.
-   [ ] Reject unauthorized fields such as `role`, `isAdmin`, `ownerId`,
    `tenantId`, `permissions`, and billing/status fields.
-   [ ] Ensure authorization is applied consistently to background jobs,
    exports, WebSocket events, GraphQL resolvers, and server actions.

**Expected fixes:** reusable `requireAuth`/`requireRole`/ownership
policies; object-scoped queries; explicit DTO/schema allowlists;
negative authorization tests.

# Phase 4 --- Multi-Tenant Isolation

-   [ ] Determine whether the application is multi-tenant and identify
    the authoritative tenant membership model.
-   [ ] Derive tenant context from a verified session/token and trusted
    membership record, not arbitrary request input.
-   [ ] Ensure every tenant-owned query, join, mutation, export, file
    operation, and background job is tenant-scoped.
-   [ ] Ensure tenant IDs cannot be changed through request bodies,
    query strings, headers, or mass assignment.
-   [ ] Validate that JWT tenant claims match the user's active and
    authorized membership.
-   [ ] Review tenant switching and ensure switching requires explicit
    authorization.
-   [ ] Enable and test database RLS for tenant-owned tables where
    supported.
-   [ ] Review both RLS `USING` and `WITH CHECK` rules for SELECT,
    INSERT, UPDATE, and DELETE.
-   [ ] Ensure privileged database/service-role connections do not
    unintentionally bypass required tenant checks.
-   [ ] Test nested resources, joins, search, analytics, reports,
    exports, storage paths, and realtime subscriptions for cross-tenant
    leaks.
-   [ ] Avoid request-specific user/tenant state in global variables,
    module-level mutable objects, singleton fields, or shared caches.
-   [ ] Ensure cache keys include the correct user/tenant scope and
    private responses cannot be reused across users.
-   [ ] Run concurrent tests with at least two tenants to detect context
    contamination.

**Required tests:** Tenant A cannot read, update, delete, export,
subscribe to, or download Tenant B data---even when IDs and tenant
parameters are manipulated.

# Phase 5 --- Database Security and Injection Prevention

-   [ ] Inventory every database access path, including raw SQL, ORM,
    stored procedures, search, reports, and migrations.
-   [ ] Verify all untrusted values use parameterized queries or safe
    ORM APIs.
-   [ ] Flag SQL string concatenation/interpolation involving
    user-controlled values.
-   [ ] Allowlist dynamic identifiers such as table names, column names,
    and sort directions; parameters generally cannot bind SQL
    identifiers.
-   [ ] Review NoSQL filters and prevent user-controlled query operators
    or object injection.
-   [ ] Avoid shell/OS command construction from untrusted input; use
    safe process APIs with fixed executable and argument arrays when
    unavoidable.
-   [ ] Validate request bodies, route parameters, query strings,
    headers, and uploaded metadata on the server.
-   [ ] Use strict schemas (e.g. Zod or equivalent) and reject
    unexpected fields.
-   [ ] Apply least-privilege database accounts; separate
    migration/admin credentials from runtime credentials.
-   [ ] Ensure the runtime account does not have unnecessary schema,
    role, extension, or destructive privileges.
-   [ ] Configure connection pool limits, statement/query timeouts, and
    request deadlines.
-   [ ] Use transactions for multi-step operations that must be atomic.
-   [ ] Review database error handling so SQL, schema names, and
    connection details are not returned to clients.

**Expected fixes:** parameterized queries, safe ORM calls, schema
validation, least-privilege DB roles, query timeouts, and safe error
mapping.

# Phase 6 --- OWASP and Application Vulnerability Review

## Injection

-   [ ] SQL injection.
-   [ ] NoSQL/operator injection.
-   [ ] OS command injection.
-   [ ] Template injection and unsafe expression evaluation.
-   [ ] LDAP or other injection types if those systems are used.

## XSS and browser security

-   [ ] Prefer framework output escaping.
-   [ ] Review `dangerouslySetInnerHTML`, `innerHTML`, `outerHTML`,
    `document.write`, `eval`, and dynamic script creation.
-   [ ] Sanitize untrusted HTML/Markdown with a maintained sanitizer
    (e.g. DOMPurify) only where HTML rendering is required.
-   [ ] Validate URL schemes and prevent `javascript:` or unsafe `data:`
    URLs in links/media.
-   [ ] Review stored, reflected, and DOM-based XSS paths.

## CSRF

-   [ ] Identify cookie-authenticated state-changing endpoints.
-   [ ] Use CSRF tokens and/or strict Origin validation as appropriate.
-   [ ] Set appropriate `SameSite` cookie attributes.
-   [ ] Do not use GET for state-changing operations.
-   [ ] Verify CORS is not incorrectly treated as CSRF protection.

## SSRF

-   [ ] Inventory server-side fetch, URL preview, webhook, import, image
    proxy, and document retrieval features.
-   [ ] Allowlist destinations where feasible.
-   [ ] Block loopback, private, link-local, reserved, and cloud
    metadata addresses.
-   [ ] Revalidate DNS resolution and redirects to prevent DNS rebinding
    and redirect-based bypasses.
-   [ ] Restrict protocols to required schemes (normally HTTPS).
-   [ ] Apply outbound network controls and strict
    timeouts/response-size limits.

## XXE and deserialization

-   [ ] Disable DTD and external entity resolution in XML parsers.
-   [ ] Avoid unsafe native/object deserialization for untrusted data.
-   [ ] Validate serialized data against strict schemas and size/depth
    limits.

## Redirects and cryptography

-   [ ] Allowlist redirect destinations; do not redirect to arbitrary
    user-provided URLs.
-   [ ] Use modern TLS and authenticated encryption for sensitive stored
    data where needed.
-   [ ] Use secure password hashing, random generation, and key
    management.
-   [ ] Avoid hardcoded salts, static IVs, obsolete algorithms, and
    custom cryptography.
-   [ ] Define key rotation and access controls.

# Phase 7 --- API Security, Rate Limiting, and Abuse Prevention

-   [ ] Inventory all API endpoints and HTTP methods.
-   [ ] Apply authentication and authorization per endpoint.
-   [ ] Add rate limits to login, OTP, password reset, registration,
    search, exports, uploads, expensive writes, and AI endpoints.
-   [ ] Use identity-aware limits (user/account/API key) plus IP-based
    controls where appropriate.
-   [ ] Add throttling, abuse detection, and lockout/backoff strategies
    without enabling easy denial-of-service against victims.
-   [ ] Limit request body size, header size, upload size, response
    size, and execution time.
-   [ ] Enforce pagination and a maximum page size of 50 unless a
    documented requirement justifies another cap.
-   [ ] For GraphQL, enforce depth, complexity/cost, resolver timeouts,
    and pagination limits.
-   [ ] Protect debug, test, metrics, admin, and internal endpoints.
-   [ ] Avoid excessive response fields; return only necessary data.
-   [ ] Validate webhook signatures and prevent replay where relevant.
-   [ ] Add idempotency controls for sensitive retryable operations
    where appropriate.
-   [ ] Ensure rate limits work across multiple application instances
    (shared store or gateway), not only process memory.

# Phase 8 --- CORS, CSP, HTTP Headers, and Cache Safety

-   [ ] Use an explicit production origin allowlist; do not use `*` for
    sensitive APIs.
-   [ ] Never combine wildcard origins with credentialed CORS.
-   [ ] Do not reflect arbitrary Origin values.
-   [ ] Restrict methods and request headers to those actually needed.
-   [ ] Set `Vary: Origin` when origin-dependent responses are used.
-   [ ] Configure a restrictive Content Security Policy; avoid broad
    `unsafe-inline`/`unsafe-eval` allowances where possible.
-   [ ] Enable HTTPS and HSTS after confirming HTTPS deployment is
    correct.
-   [ ] Set `X-Content-Type-Options: nosniff`.
-   [ ] Set an appropriate `Referrer-Policy`.
-   [ ] Configure clickjacking protection through CSP `frame-ancestors`
    (and legacy headers if needed).
-   [ ] Set `Cache-Control: no-store` for sensitive authenticated
    responses.
-   [ ] Configure CDN, reverse proxy, browser, and application caches to
    prevent cross-user/tenant response reuse.
-   [ ] Ensure service workers do not cache private API responses
    improperly.

# Phase 9 --- File Uploads and Download Security

-   [ ] Enforce validation on the server; client-side checks are
    usability only.
-   [ ] Validate actual file signatures/magic numbers, not only
    extension or declared MIME type.
-   [ ] Allowlist only necessary file formats.
-   [ ] Set strict upload limits at reverse proxy and application layers
    (e.g. 5 MB if appropriate).
-   [ ] Reject oversized payloads before loading them fully into memory.
-   [ ] Generate random storage names; do not trust user-provided paths
    or filenames.
-   [ ] Store uploads outside executable web roots or in private object
    storage.
-   [ ] Disable script execution in upload storage.
-   [ ] Set safe `Content-Type` and `Content-Disposition` headers on
    downloads.
-   [ ] Prevent path traversal, archive extraction traversal,
    decompression bombs, and symlink attacks.
-   [ ] Scan files for malware when justified by file types and threat
    model.
-   [ ] Enforce ownership/tenant authorization on every upload, preview,
    and download.
-   [ ] Use short-lived signed download URLs where appropriate and scope
    them to the correct object.

# Phase 10 --- Frontend, React/Next.js, and Client Runtime

-   [ ] Check hooks (`useEffect`, `useMemo`, custom hooks, query hooks)
    for incorrect dependencies and recursive state updates.
-   [ ] Detect unbounded polling, retries, subscriptions, and repeated
    network calls.
-   [ ] Ensure timers, event listeners, observers, sockets, and
    subscriptions are cleaned up.
-   [ ] Prevent sensitive tokens and secrets from entering browser
    storage, logs, analytics, or client bundles.
-   [ ] Ensure frontend route guards are backed by server-side
    authorization.
-   [ ] Validate all client-originated data again on the server.
-   [ ] Review third-party scripts, analytics, SDKs, and supply-chain
    exposure.
-   [ ] Review source maps and production build artifacts.
-   [ ] Ensure no project root or private files are exposed through
    static hosting.

# Phase 11 --- LLM/AI Features (if applicable)

-   [ ] Treat prompts, retrieved documents, web content, and uploaded
    files as untrusted.
-   [ ] Test direct and indirect prompt injection.
-   [ ] Keep system instructions and secrets out of user-controlled
    context.
-   [ ] Do not rely on model output for authorization decisions.
-   [ ] Validate tool/function arguments with strict schemas.
-   [ ] Require server-side authorization before every tool action.
-   [ ] Apply least privilege to tools and external integrations.
-   [ ] Restrict destinations, filesystem access, commands, and data
    scopes available to agents.
-   [ ] Limit input size, output size, token budgets, concurrency,
    retries, and per-user spending.
-   [ ] Prevent cross-user retrieval/memory contamination.
-   [ ] Redact sensitive data from prompts, traces, and logs where
    possible.

# Phase 12 --- Server and Infrastructure Hardening

-   [ ] Disable debug mode and verbose production errors.
-   [ ] Block public access to `.env`, `.git`, private keys, backups,
    source/config files, and internal paths.
-   [ ] Do not serve the entire project root as static content.
-   [ ] Expose only required network ports.
-   [ ] Restrict database and admin access to trusted networks.
-   [ ] Run services as non-root/least-privilege users.
-   [ ] Remove default credentials, unused services, sample routes, and
    development tools.
-   [ ] Secure container images, runtime users, filesystem permissions,
    and capabilities.
-   [ ] Review reverse proxy, TLS termination, trusted proxy, and
    forwarded-header configuration.
-   [ ] Patch OS, runtime, framework, and server dependencies.
-   [ ] Verify production configuration differs safely from development
    configuration.

# Phase 13 --- Privacy, PII, and Data Protection

-   [ ] Identify personal and sensitive data collected, processed,
    stored, transmitted, and logged.
-   [ ] Minimize collection and retain data only as long as necessary.
-   [ ] Use TLS for data in transit.
-   [ ] Encrypt sensitive data at rest where appropriate, with
    controlled key access.
-   [ ] Never store passwords or reset/OTP secrets in plaintext.
-   [ ] Redact credentials, tokens, and unnecessary PII from logs and
    telemetry.
-   [ ] Restrict staff/admin access to personal data.
-   [ ] Define retention, deletion, export, and access procedures.
-   [ ] Review third-party processors and data-sharing flows.
-   [ ] Verify privacy notices, consent, and applicable legal
    obligations for the deployment jurisdiction.

# Phase 14 --- Logging, Monitoring, and Resource Safety

-   [ ] Use centralized error handling and consistent safe error
    responses.
-   [ ] Generate request/correlation IDs and include them in logs and
    client error responses.
-   [ ] Log security-relevant events without logging secrets or full
    sensitive payloads.
-   [ ] Monitor authentication failures, authorization denials,
    rate-limit triggers, and suspicious patterns.
-   [ ] Monitor CPU, memory, disk, database connections, queue depth,
    latency, and error rates.
-   [ ] Clean up timers, event listeners, subscriptions, streams, and
    database connections.
-   [ ] Check for memory leaks, unbounded caches, unbounded queues, and
    runaway background jobs.
-   [ ] Use heap snapshots/profiling for sustained memory growth.
-   [ ] Configure alerts and incident escalation paths.
-   [ ] Ensure logs have access controls, retention, integrity, and
    appropriate redaction.

# Phase 15 --- Dependencies and CI/CD Supply Chain

-   [ ] Run dependency vulnerability scans using the ecosystem's
    supported tools.
-   [ ] Remove unused, abandoned, or unnecessary packages.
-   [ ] Review lockfiles and pin reproducible dependency versions.
-   [ ] Review package lifecycle scripts and suspicious dependencies.
-   [ ] Run SAST and secret scanning in CI.
-   [ ] Run DAST against a safe staging deployment.
-   [ ] Scan container images and infrastructure-as-code.
-   [ ] Protect CI/CD secrets and restrict deployment permissions.
-   [ ] Require review and automated checks before production release.
-   [ ] Generate and retain a software bill of materials (SBOM) where
    practical.
-   [ ] Review dependency licenses and provenance.

# Phase 16 --- Backup, Disaster Recovery, and Availability

-   [ ] Identify primary databases, object storage, uploaded files,
    configuration, and other critical state.
-   [ ] Implement automated daily backups (or a frequency matching
    business RPO).
-   [ ] Store backups in an isolated off-site account/environment.
-   [ ] Encrypt backups and restrict access using least privilege.
-   [ ] Protect backups against ransomware, accidental deletion, and
    compromised production credentials.
-   [ ] Define Recovery Point Objective (RPO) and Recovery Time
    Objective (RTO).
-   [ ] Define retention and deletion policies.
-   [ ] Test restoration periodically in an isolated environment.
-   [ ] Verify restored data integrity and application functionality.
-   [ ] Document incident response, recovery steps, ownership, and
    escalation.
-   [ ] Do not claim disaster recovery readiness until restoration has
    been demonstrated.

# Phase 17 --- Performance and Cloud Cost Controls

-   [ ] Prevent infinite loops, recursive effects, unbounded retries,
    and excessive polling.
-   [ ] Limit expensive database queries and execution time.
-   [ ] Enforce pagination and response-size caps.
-   [ ] Set connection pool limits and request deadlines.
-   [ ] Apply caching only when authorization and privacy boundaries are
    preserved.
-   [ ] Limit file size, storage use, and upload frequency.
-   [ ] Apply AI token, concurrency, request, and spending limits if
    applicable.
-   [ ] Set cloud quotas, budget alerts, and anomaly detection.
-   [ ] Review scheduled jobs and queue retry/dead-letter behavior.
-   [ ] Load-test critical endpoints in staging.
-   [ ] Estimate expected cost at normal, peak, and abusive traffic
    levels.

# Phase 18 --- Security Testing and Release Gate

-   [ ] Test SQL/NoSQL/command injection with safe test payloads.
-   [ ] Test stored, reflected, and DOM XSS.
-   [ ] Test CSRF on cookie-authenticated state-changing routes.
-   [ ] Test SSRF protections, redirects, DNS changes, and internal IP
    blocking.
-   [ ] Test IDOR/BOLA with at least two users and two tenants.
-   [ ] Test role escalation and default-deny behavior.
-   [ ] Test JWT validation, expiry, revocation, refresh rotation, and
    OAuth state.
-   [ ] Test rate limits and brute-force protections.
-   [ ] Test malicious file types, spoofed MIME types, oversized files,
    and path traversal.
-   [ ] Test CORS, CSP, security headers, and private cache behavior.
-   [ ] Test concurrent requests for cross-user/tenant context leakage.
-   [ ] Test pagination, query timeouts, connection pool exhaustion, and
    GraphQL complexity if applicable.
-   [ ] Verify no secrets in current files, Git history, build
    artifacts, or client bundles.
-   [ ] Verify backup restoration.
-   [ ] Run regression tests after fixes.
-   [ ] Resolve all Critical and High findings before release unless a
    documented, formally approved exception exists.
-   [ ] Document residual risks, owners, due dates, and compensating
    controls.
-   [ ] Obtain explicit release approval.

------------------------------------------------------------------------

# Severity Classification

  -----------------------------------------------------------------------
  Severity                Typical examples        Release decision
  ----------------------- ----------------------- -----------------------
  **Critical**            Remote code execution,  **Block deployment**
                          unauthenticated admin   
                          access, unrestricted    
                          database exposure,      
                          major cross-tenant      
                          breach, active secret   
                          compromise              

  **High**                SQL injection,          **Block deployment**
                          IDOR/BOLA exposing      
                          sensitive records,      
                          privilege escalation,   
                          exposed service-role    
                          keys, unsafe upload     
                          leading to code         
                          execution               

  **Medium**              Missing rate limits,    Fix before release
                          weak security headers,  where feasible;
                          incomplete validation,  otherwise formally
                          insufficient            accept risk
                          monitoring, limited     
                          information disclosure  

  **Low**                 Minor hardening gaps,   Track with owner and
                          low-impact metadata     deadline
                          disclosure,             
                          documentation or        
                          defense-in-depth        
                          improvements            
  -----------------------------------------------------------------------

Severity must be based on exploitability, impact, exposure, and
available compensating controls---not merely on the category name.

# Required Audit Deliverables

The agent must produce these files (or clearly equivalent sections in
one report):

1.  `architecture-inventory.md` --- architecture, trust boundaries, data
    flows, endpoints, and assumptions.
2.  `security-findings.md` --- findings grouped by Critical, High,
    Medium, and Low.
3.  `endpoint-authorization-matrix.md` --- route, method, authentication
    requirement, role/permission, ownership rule, tenant scope, rate
    limit.
4.  `secrets-audit.md` --- locations and types of possible secrets,
    fully redacted; include Git-history status.
5.  `database-rls-audit.md` --- tables, RLS status, policies,
    tenant/ownership conditions, and test evidence.
6.  `security-test-results.md` --- tests executed, test environment,
    results, and limitations.
7.  `remediation-plan.md` --- prioritized fixes, affected files,
    implementation notes, regression tests, and owners.
8.  `production-readiness.md` --- final release decision and unresolved
    risks.

## Required format for every finding

``` text
ID: SEC-###
Title:
Severity: Critical | High | Medium | Low
Status: Confirmed | Suspected | Not Verified
Affected component:
Affected file/route/function:
Evidence:
Preconditions:
Exploit scenario:
Impact:
Recommended remediation:
Example code/config fix:
Regression test:
References:
```

## Required format for every checklist item

``` text
Status: PASS | FAIL | PARTIAL | NOT APPLICABLE | NOT VERIFIED
Evidence:
Files/routes inspected:
Test performed:
Result:
Remaining gap:
```

# Final Production Decision

The agent must end the report with exactly one decision:

-   **BLOCK:** Any unresolved Critical or High issue, or a material
    security area that could not be verified.
-   **CONDITIONAL GO:** No known Critical/High issue, but documented
    Medium/Low risks or operational controls remain with explicit
    owners, deadlines, and approval.
-   **GO:** Required security controls have been inspected and tested;
    no unresolved Critical/High findings remain; backup restoration and
    release checks have passed.

A checklist marked complete is not proof of security. Evidence and tests
are mandatory. If the agent cannot inspect a service, secret store,
cloud configuration, database policy, or production header, it must
report `NOT VERIFIED` rather than assume it is secure.
