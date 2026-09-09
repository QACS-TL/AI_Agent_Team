# JWT Authentication Architecture (ASP.NET Core Web API, Single Internal Issuer)

## 1. Architecture Overview

### Goals
- Provide robust JWT-based authentication and authorization for internal and external API consumers.
- Centralize token issuance and validation policy.
- Enable secure key rotation, tenant/environment isolation, and least-privilege access.

### Logical Components
1. **API Gateway / Edge (optional but recommended)**
   - TLS termination, rate limiting, WAF, request correlation.
   - Forwards bearer tokens to downstream API services.

2. **Auth Service (Internal Issuer)**
   - Authenticates users/services (password, federated upstream, mTLS, or API key bridge).
   - Issues JWT access tokens and optional refresh tokens.
   - Exposes JWKS endpoint for signing key discovery.

3. **Resource APIs (ASP.NET Core Web API)**
   - Validate incoming JWTs using issuer metadata/JWKS.
   - Apply authorization policies (roles/scopes/claims/resource rules).

4. **Identity Store**
   - User credentials (if local auth), service principals, roles, claims, and revocation metadata.

5. **Key Management**
   - Secure private signing keys (HSM/Key Vault strongly recommended).
   - Publishes public keys via JWKS with versioned `kid`.

6. **Observability & Security Monitoring**
   - Authentication/authorization audit logs.
   - Token failure telemetry (expired, invalid signature, invalid audience).

### Reference Flow
1. Client authenticates with Auth Service.
2. Auth Service issues signed JWT (`iss`, `aud`, `exp`, `nbf`, `sub`, `jti`, scopes/roles).
3. Client calls Resource API with `Authorization: Bearer <token>`.
4. API validates token signature/claims via JWT bearer middleware.
5. Authorization policies evaluate claims and resource constraints.
6. API returns result or `401/403`.

---

## 2. Required Services

### A. Authentication Issuer Service
- Endpoints:
  - `/connect/token` (or equivalent) for token issuance.
  - `/connect/refresh` for refresh exchange (optional).
  - `/.well-known/jwks.json` for key distribution.
- Responsibilities:
  - Identity verification.
  - Token minting and signing.
  - Refresh token lifecycle management.

### B. Key Management Service
- Secure key generation/storage.
- Key versioning and rotation workflow.
- Public key exposure via JWKS.

### C. User/Principal Service
- Credential validation.
- Claims/roles/scope assignment.
- Account state checks (disabled/locked/tenant status).

### D. Revocation/Session Service (Recommended)
- Track revoked refresh tokens and high-risk access token JTIs.
- Support emergency invalidation events.

### E. API Authorization Service Layer
- Encapsulate domain authorization logic beyond simple role checks.
- Example: ownership checks, environment-bound permissions.

### F. Audit & Monitoring Service
- Immutable security logs.
- Alerting on anomalies (replay, brute force, signature failures, clock skew spikes).

---

## 3. Interfaces

> The following interfaces define boundaries for ASP.NET Core implementation.

### Token Issuance
- `ITokenService`
  - `Task<TokenResult> IssueAccessTokenAsync(TokenRequest request, CancellationToken ct)`
  - `Task<TokenResult> RefreshAccessTokenAsync(RefreshRequest request, CancellationToken ct)`

- `ISigningCredentialsProvider`
  - `Task<SigningCredentials> GetActiveSigningCredentialsAsync(CancellationToken ct)`
  - `Task<IReadOnlyCollection<JsonWebKey>> GetPublicKeysAsync(CancellationToken ct)`

### Identity & Claims
- `IUserAuthenticationService`
  - `Task<AuthPrincipal?> AuthenticateAsync(CredentialInput input, CancellationToken ct)`

- `IClaimsFactory`
  - `Task<IReadOnlyCollection<Claim>> BuildClaimsAsync(AuthPrincipal principal, CancellationToken ct)`

### Revocation & Session Control
- `ITokenRevocationService`
  - `Task<bool> IsRevokedAsync(string jti, CancellationToken ct)`
  - `Task RevokeAsync(string jti, DateTimeOffset expiresAt, CancellationToken ct)`

### Authorization Extension
- `IResourceAuthorizationService`
  - `Task<bool> AuthorizeAsync(ClaimsPrincipal user, string resourceId, string action, CancellationToken ct)`

### Auditing
- `IAuthenticationAuditService`
  - `Task LogTokenIssuedAsync(TokenIssuedEvent evt, CancellationToken ct)`
  - `Task LogAuthFailureAsync(AuthFailureEvent evt, CancellationToken ct)`

---

## 4. Configuration Requirements

### JWT Issuer Configuration
- `Jwt:Issuer` (e.g., `https://auth.internal.company`)
- `Jwt:Audience` (single or multiple audiences)
- `Jwt:AccessTokenLifetimeMinutes` (short-lived; typically 5-15)
- `Jwt:RefreshTokenLifetimeDays` (if enabled)
- `Jwt:RequireHttpsMetadata` = `true`
- `Jwt:ClockSkewSeconds` (minimal, e.g., <= 60)

### Key Configuration
- `Jwt:SigningKey:Source` (`KeyVault`, `HSM`, `CertificateStore`)
- `Jwt:SigningKey:ActiveKid`
- `Jwt:SigningKey:RotationIntervalDays`
- `Jwt:SigningKey:AllowedAlgorithms` (`RS256`/`ES256`; avoid symmetric shared secrets across services)

### Validation Configuration (Resource APIs)
- Validate:
  - Issuer (`ValidateIssuer = true`)
  - Audience (`ValidateAudience = true`)
  - Lifetime (`ValidateLifetime = true`)
  - Signature (`ValidateIssuerSigningKey = true`)
- Enforce allowed algorithms explicitly.
- Reject unsigned tokens and `alg=none`.

### Environment & Secrets
- No secrets in source control.
- Store private keys and connection secrets in secure secret provider.
- Separate issuer, audience, and keys per environment.

---

## 5. Dependency Injection Design (ASP.NET Core)

### Registration Structure
- `services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme).AddJwtBearer(...)`
- `services.AddAuthorization(options => { /* policies */ })`
- `services.AddScoped<IUserAuthenticationService, UserAuthenticationService>()`
- `services.AddScoped<IClaimsFactory, ClaimsFactory>()`
- `services.AddScoped<ITokenService, JwtTokenService>()`
- `services.AddSingleton<ISigningCredentialsProvider, KeyVaultSigningCredentialsProvider>()`
- `services.AddScoped<ITokenRevocationService, TokenRevocationService>()`
- `services.AddScoped<IResourceAuthorizationService, ResourceAuthorizationService>()`
- `services.AddScoped<IAuthenticationAuditService, AuthenticationAuditService>()`

### Lifetime Guidance
- **Singleton**: key providers/caches that are thread-safe.
- **Scoped**: request-bound business services and authorization logic.
- **Transient**: lightweight stateless helpers when needed.

### Middleware Pipeline
1. Exception handling / security headers middleware.
2. Routing.
3. `UseAuthentication()`.
4. `UseAuthorization()`.
5. Endpoint mapping.

### Policy-Based Authorization
- Define granular policies by scope/role/claim.
- Use custom `IAuthorizationHandler` for resource-specific checks.

---

## 6. Security Considerations

1. **Short-Lived Access Tokens**
   - Minimize blast radius of token leakage.

2. **Asymmetric Signing Keys**
   - Keep private keys only in issuer.
   - APIs validate using public keys only.

3. **Key Rotation & `kid` Management**
   - Support overlapping keys during rotation window.
   - Ensure cache invalidation strategy for JWKS consumers.

4. **Strict Claim Validation**
   - Enforce issuer/audience/lifetime/signature/algorithm checks.
   - Validate tenant/context claims for multi-tenant scenarios.

5. **Replay Mitigation**
   - Use `jti` and revocation checks for sensitive operations.
   - Consider DPoP or mTLS-bound tokens for high-security APIs.

6. **Refresh Token Security**
   - Store hashed refresh tokens server-side.
   - One-time use with rotation and reuse-detection.

7. **Transport Security**
   - Enforce HTTPS/TLS 1.2+ everywhere.
   - Deny tokens over non-secure channels.

8. **Operational Hardening**
   - Rate limit token endpoints.
   - Brute-force protections and account lockouts.
   - Centralized audit logging and anomaly detection.

9. **Data Minimization**
   - Avoid sensitive PII in JWT payload.
   - Keep claims minimal and purpose-bound.

---

## 7. Risks

1. **Key Compromise Risk**
   - Impact: full token forgery potential.
   - Mitigation: HSM/Key Vault, strict access controls, emergency key rollover runbook.

2. **Clock Drift Across Systems**
   - Impact: valid tokens rejected or expired tokens accepted.
   - Mitigation: NTP sync, minimal skew configuration, monitor drift.

3. **Over-Permissive Claims/Policies**
   - Impact: privilege escalation.
   - Mitigation: least privilege, policy reviews, automated authorization tests.

4. **Revocation Gaps for Stateless JWTs**
   - Impact: compromised token remains usable until expiry.
   - Mitigation: short TTL, selective JTI blocklist, high-risk endpoint introspection.

5. **Rotation Coordination Failures**
   - Impact: auth outages during key transitions.
   - Mitigation: dual-key validation window, staged rollout, cache TTL tuning.

6. **Token Leakage in Logs/Telemetry**
   - Impact: credential exposure.
   - Mitigation: redact headers/payloads, secure log sinks, access auditing.

7. **Single Issuer Availability Dependency**
   - Impact: issuance outage affects new sessions.
   - Mitigation: HA deployment, health probes, disaster recovery, queue/backoff on clients.

---

## Recommended Next Steps
1. Implement the interfaces and wire up JWT bearer authentication in a shared auth library.
2. Add integration tests for token validation, expiry, audience mismatch, and policy checks.
3. Implement key rotation runbook and incident response procedure.
4. Add security dashboards for `401/403`, invalid signature rates, and revocation events.
