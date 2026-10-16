# Security

Language-neutral security architecture. Pseudocode conventions are in [SKILL.md](../SKILL.md#pseudocode-conventions). Use your framework's built-in security features and well-maintained libraries — never invent your own crypto or token formats. Cross-check against the [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/) and [OWASP Top 10](https://owasp.org/www-project-top-ten/).

## Table of Contents

1. [Authentication](#authentication)
2. [Password Storage](#password-storage)
3. [Tokens and Sessions](#tokens-and-sessions)
4. [Authorization](#authorization)
5. [Input Validation and Output Encoding](#input-validation-and-output-encoding)
6. [Injection Prevention](#injection-prevention)
7. [Rate Limiting](#rate-limiting)
8. [Secrets Management](#secrets-management)
9. [Transport and Headers](#transport-and-headers)
10. [Supply Chain](#supply-chain)

---

## Authentication

- **Prefer delegating identity** to a proven provider (OIDC/OAuth 2.x via Keycloak, Auth0, Okta, Entra ID, Cognito, Firebase Auth, Zitadel, …) over building login yourself. You still validate tokens and authorise.
- If you run your own: password + **MFA** (TOTP/WebAuthn/passkeys), account lockout or progressive delays, secure reset flows with single-use, short-lived tokens, and generic error messages ("invalid email or password").
- **Authentication happens at the edge** (middleware/filter) and produces a `Principal` that flows into use cases.

---

## Password Storage

```
hashPassword(plain) -> string {
  return Argon2id.hash(plain)              // memory-hard; library chooses salt and encodes parameters
}

verifyPassword(plain, stored) -> bool {
  ok = Argon2id.verify(plain, stored)      // constant-time comparison inside the library
  if ok and Argon2id.needsRehash(stored):  // parameters upgraded since this hash was made
    scheduleRehash(plain)
  return ok
}
```

- Use **Argon2id** (preferred), **scrypt**, or **bcrypt** with a sensible cost factor — via the platform's standard library or a well-known package (Spring Security `PasswordEncoder`, ASP.NET Identity, `argon2`/`bcrypt` packages for Node, `argon2-cffi`/`passlib`/`pwdlib` for Python, `golang.org/x/crypto/argon2` or `bcrypt`, the `argon2` crate, PHP `password_hash()`, Rails `has_secure_password`, `argon2_elixir`).
- Never use fast hashes (MD5, SHA-*) for passwords. Never log passwords or hashes.
- Check new passwords against breached-password lists; prefer length (≥ 12) over composition rules.

---

## Tokens and Sessions

| Mechanism | Fits | Key points |
|---|---|---|
| **Server-side session + cookie** | Traditional web apps, BFFs | `HttpOnly`, `Secure`, `SameSite=Lax/Strict` cookie; rotate session ID on login; CSRF protection for cookie-authenticated state changes |
| **Short-lived access token (JWT) + refresh token** | APIs, SPAs via BFF, mobile, service-to-service | Access token minutes-long; refresh token rotated and revocable; store refresh tokens server-side or in secure device storage |
| **Opaque tokens + introspection** | When instant revocation matters | Extra lookup per request (cache it briefly) |

```
// Verifying a JWT (always via a library)
verifyAccessToken(token) -> Claims {
  claims = Jwt.verify(token,
    keys        = jwks.current(),          // fetched from the issuer and cached; supports key rotation
    algorithms  = ["RS256", "ES256"],      // explicit allow-list; never accept "none"
    issuer      = config.oidcIssuer,
    audience    = config.apiAudience,
    clockSkew   = 30 seconds)              // fails on bad signature, wrong iss/aud, expired exp/nbf
  if claims.sub is blank: fail Unauthenticated("Missing subject")
  return claims
}
```

- Prefer **asymmetric signing** (RS256/ES256/EdDSA) so services verify without holding the signing key. HS256 is acceptable only when the issuer and verifier are the same trusted service and the secret is long and random.
- Keep JWTs small; don't put sensitive data in them (they're only signed, not encrypted).
- **Browser apps**: don't keep long-lived tokens in `localStorage` (XSS-readable). Use a BFF with HttpOnly cookies, or in-memory access tokens with refresh via secure cookie.
- **Mobile/desktop**: store tokens in the platform keystore (iOS Keychain, Android Keystore, Windows Credential Locker, macOS Keychain); use OAuth Authorization Code + PKCE.

---

## Authorization

Authentication says *who*; authorization says *may they do this to this object*.

```
// Coarse-grained at the edge
route DELETE "/v1/orders/{id}" requires scope "orders:write"

// Fine-grained in the application layer (object-level — prevents IDOR / BOLA)
class CancelOrderUseCase(orders: OrderRepository, policy: OrderPolicy, clock) {
  execute(principal: Principal, orderId: OrderId) {
    order = orders.get(orderId) ?? fail NotFound("Order", orderId)
    if not policy.canCancel(principal, order): fail Forbidden("Not allowed to cancel this order")
    order.cancel(clock)
    orders.update(order)
  }
}

class OrderPolicy {
  canCancel(p: Principal, order: Order) -> bool {
    return p.hasRole("support")
        or (p.userId == order.customerId and order.status == PENDING)
  }
}
```

| Model | Use when |
|---|---|
| **RBAC** (roles → permissions) | Most business apps; keep roles coarse and map them to fine permissions in code |
| **ABAC / policy-based** (attributes of user, resource, context) | Rules depend on ownership, tenant, time, amount |
| **ReBAC** (relationships, Zanzibar-style: OpenFGA, SpiceDB, Permify) | Sharing/collaboration graphs (docs, folders, orgs) |
| **Policy engines** (OPA/Rego, Cedar, Casbin) | Centralised, auditable policies across many services |

Rules: **deny by default**; check on **every** request server-side (never trust the client UI to hide things); scope every query by tenant in multi-tenant systems (row-level security or mandatory tenant filters); return 404 rather than 403 when revealing existence is itself a leak.

---

## Input Validation and Output Encoding

```
record UserProfileUpdate {
  name: string   (1..100 chars, trimmed, no control characters)
  bio:  string?  (max 1000 chars)
  website: string? (absolute URL, scheme in ["https"])
}
```

- **Validate at the boundary** with an allow-list mindset: types, lengths, ranges, formats, enums. Reject rather than "fix" invalid input.
- **Limit request size** (body, headers, array lengths, JSON depth) to prevent resource exhaustion.
- **Encode output for its context**, rather than "sanitising" input for every possible future context:
  - HTML: use the template engine's auto-escaping (React/Vue/Angular, Razor, Thymeleaf, Jinja, ERB, Blade, HEEx all escape by default). Avoid raw-HTML escape hatches (`dangerouslySetInnerHTML`, `v-html`, `|safe`, `raw`, `{!! !!}`).
  - If you must accept rich text, sanitise with a vetted HTML sanitiser using an allow-list of tags/attributes (DOMPurify, OWASP Java HTML Sanitizer, HtmlSanitizer, bleach successors like `nh3`, `bluemonday`, `ammonia`, `HTMLPurifier`, Rails `sanitize`).
- **File uploads**: check size and type by content (magic bytes), generate your own filenames, store outside the web root or in object storage, serve with `Content-Disposition` and correct `Content-Type`, scan if risk warrants.
- **SSRF**: when fetching user-supplied URLs, allow-list hosts/schemes, block private/link-local/metadata IP ranges after DNS resolution, and disable redirects or re-validate them.
- **Deserialisation**: never deserialise untrusted data into arbitrary types (Java native serialisation, Python `pickle`, PHP `unserialize`, YAML with object tags); use data-only formats with schema validation.

---

## Injection Prevention

```
// ✅ Parameterised query — the driver sends data separately from SQL
db.query("SELECT * FROM users WHERE email = $1", [userInput])

// ❌ String concatenation — SQL injection
db.query("SELECT * FROM users WHERE email = '" + userInput + "'")

// Dynamic identifiers (sort column, table) can't be parameters — map from an allow-list
sortColumn = { "created": "created_at", "name": "name" }[req.query.sort] ?? "created_at"
```

The same principle applies to every interpreter: **OS commands** (pass argument arrays, never a shell string), **LDAP**, **NoSQL** (no operator injection from raw JSON like `{"$ne": null}`), **template engines** (never render user input as a template), **log injection** (structured logging escapes newlines), **header injection** (reject CR/LF).

---

## Rate Limiting

Protect authentication endpoints, expensive operations, and the API overall.

| Algorithm | Behaviour |
|---|---|
| Fixed window | Simple counter per window; allows bursts at window edges |
| Sliding window | Smoother; small extra cost |
| Token bucket | Allows controlled bursts; refills at a steady rate — a good default |

```
// Token bucket in a shared store (implemented atomically, e.g. as a Redis Lua script)
allow(key, capacity = 100, refillPerSecond = 100 / 3600) -> (allowed: bool, retryAfter: Duration) {
  atomically {
    (tokens, last) = store.get(key) ?? (capacity, now())
    tokens = min(capacity, tokens + (now() - last).seconds × refillPerSecond)
    if tokens < 1: return (false, secondsUntil(1 token))
    store.set(key, (tokens - 1, now()), ttl = timeToFull(capacity, refillPerSecond))
    return (true, 0)
  }
}

middleware rateLimit(req, next) {
  key = "rl:" + (req.principal?.id ?? clientIp(req))     // clientIp honours trusted proxy headers only
  (ok, retryAfter) = limiter.allow(key)
  if not ok: return respond(429, problem("Rate limit exceeded"), headers = { "Retry-After": retryAfter.seconds })
  return next(req)
}
```

- Tighter limits on login, password reset, signup, OTP verification (per account **and** per IP).
- Prefer limiting at the gateway/edge (API gateway, CDN/WAF, reverse proxy) and keep app-level limits for business-specific quotas.
- Libraries: Bucket4j/Resilience4j, ASP.NET Core rate limiting middleware, `rate-limiter-flexible`, `slowapi`/`limits`, `golang.org/x/time/rate`, `governor`/`tower`, Laravel `RateLimiter`, Rack::Attack, `Hammer`/`PlugAttack`.

---

## Secrets Management

- **Never commit secrets.** Use `.env.example` with placeholders; add secret scanning (gitleaks, trufflehog, GitHub secret scanning) to CI and pre-commit hooks.
- Load secrets at runtime from a secret manager (Vault, AWS Secrets Manager/SSM, GCP Secret Manager, Azure Key Vault, Doppler, 1Password, Kubernetes Secrets backed by an external store/sealed secrets).
- **Rotate** regularly and on any suspected exposure; prefer short-lived, dynamically issued credentials and workload identity (IAM roles, workload identity federation) over static keys.
- **Never log secrets, tokens, passwords, or full PII.** Redact in the logging layer as a safety net.
- Encrypt sensitive data at rest (database/storage encryption, plus application-level encryption with a KMS for especially sensitive fields).

---

## Transport and Headers

- **HTTPS everywhere** (TLS 1.2+, prefer 1.3), HSTS on public domains; mTLS for service-to-service where feasible.
- **Security headers** for web responses: `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, and `frame-ancestors` in CSP (or `X-Frame-Options`).
- **CORS**: explicit allow-list of origins; never reflect arbitrary `Origin` with `Access-Control-Allow-Credentials: true`.
- **Errors** never include stack traces, SQL, or internal hostnames in responses.

---

## Supply Chain

- Pin dependencies with lockfiles; update regularly with automation (Dependabot, Renovate).
- Scan dependencies and images (`npm audit`, `pip-audit`, `govulncheck`, `cargo audit`, OWASP Dependency-Check, `dotnet list package --vulnerable`, `composer audit`, `bundler-audit`, `mix deps.audit`, Trivy, Grype, Snyk).
- Run SAST (CodeQL, Semgrep, language linters with security rules) in CI.
- Build in CI from clean environments; sign artifacts/images (Sigstore/cosign) and generate SBOMs where required.
