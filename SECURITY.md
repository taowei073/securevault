# Security

SecureVault is a portfolio project. This document holds its threat model and,
later, the results of security testing and scanning.

**Status:** draft threat model, written before the app code exists. It will be
updated as controls are built and tested.

## Threat model (STRIDE)

### Assets

| Asset | Why it matters |
|---|---|
| Note contents | The private data users trust the app with |
| Credentials (password hashes) | A leak enables account takeover elsewhere |
| JWT signing key | Whoever has the key can impersonate any user |
| Access and refresh tokens | A stolen token is a stolen session |
| User roles and lock state | Changing them grants admin or bypasses lockout |
| Audit log | Evidence of who did what; useless if it can be altered or skipped |

### Actors

| Actor | Can do |
|---|---|
| Anonymous user | Register, log in |
| Regular user | Manage own notes, read notes shared with them |
| Admin | List users, lock/unlock accounts, view the audit log. **Cannot read note contents** (least privilege). |
| Attacker | An outsider, or a logged-in user trying to exceed their rights |

### Trust boundaries

1. **Client → app:** all requests, form fields, headers, cookies and tokens are untrusted.
2. **Regular user → admin functions:** privilege boundary enforced on every admin route.
3. **App → database:** only the app talks to the database, using parameterized queries.
4. **App → environment:** secrets come from environment variables, never from the repo.

### Session and token design

Decided up front because it shapes phase 1:

- **Browser pages:** tokens live in `HttpOnly`, `Secure`, `SameSite=Strict` cookies, so
  JavaScript (and therefore XSS) can't read them. Because browsers send cookies
  automatically, every form also carries a CSRF token.
- **API clients and tests:** `Authorization: Bearer` header. Browsers never send this on
  their own, so CSRF doesn't apply.
- **Access tokens** are short-lived JWTs. **Refresh tokens** are stored server-side so they
  can be revoked on logout, and rotate on every use.

### Threats and planned mitigations

| STRIDE | Threat | Planned mitigation | OWASP |
|---|---|---|---|
| **S**poofing | Password guessing or credential stuffing | argon2 hashing, password policy, lockout after 5 failed logins, login rate limits | A07, API4 |
| **S**poofing | Forged, tampered, expired or `alg=none` JWT | Signature, expiry and algorithm checks; short-lived access tokens | API2 |
| **S**poofing | Stolen refresh token reused after logout or theft | Server-side refresh tokens revoked on logout; rotation with reuse detection (reuse of an old token revokes the whole session) | API2, A07 |
| **S**poofing | Cross-site request forgery using the session cookie | `SameSite=Strict` cookies plus a CSRF token on every form | A01 |
| **T**ampering | Mass assignment of `role` or `owner_id` | Separate request schemas that don't include these fields | API3 |
| **T**ampering | SQL injection in note fields | Pydantic validation, ORM parameterized queries | A03 |
| **T**ampering | Stored XSS in note contents | Jinja2 autoescaping, CSP; `HttpOnly` cookies so a script can't steal the session | A03 |
| **T**ampering | Altering or deleting audit records | Admins can view the audit log but not edit or delete it | A09 |
| **T**ampering | Log injection via newlines in usernames | Structured JSON logging escapes field values | A09 |
| **R**epudiation | A user or admin denies a login or admin action | Structured JSON audit log for logins, failures, admin actions and access denials | A09 |
| **I**nformation disclosure | Reading another user's note by ID (BOLA) | Ownership check on every note endpoint; return 404, not 403, so the note's existence isn't confirmed | API1, A01 |
| **I**nformation disclosure | Username enumeration; stack traces in errors | Same response for unknown user and wrong password; generic error messages | A05, A07 |
| **I**nformation disclosure | Secrets or private data in logs | Never log passwords, tokens or note contents | A09 |
| **I**nformation disclosure | Secrets leaked through the repo | Secrets only from environment variables; `.env.example` committed, `.env` ignored; private-key detection in pre-commit | A05 |
| **D**enial of service | Flooding the login endpoint | Per-IP rate limits with slowapi | API4 |
| **D**enial of service | Locking real users out by failing logins with their username | Time-based lockout (15 minutes, not permanent), per-IP rate limits, admin unlock | A07, API4 |
| **D**enial of service | Huge notes or request bodies | Maximum note size and request body limit | API4 |
| **E**levation of privilege | Regular user calls admin endpoints | Role guard on admin routes returning 403 | API5 |
| **E**levation of privilege | Shared note edited by its recipient | Sharing grants read-only access, checked on every write | API1, A01 |
| **E**levation of privilege | `INSECURE_MODE` left on outside tests | Off by default; app refuses to start with it unless `ENV=test` | A05 |
| **T** / **I** | Clickjacking, MIME sniffing, cross-origin abuse | CSP, HSTS, X-Frame-Options, X-Content-Type-Options, strict CORS | A05 |

**Trade-off:** account lockout stops password guessing, but it lets an attacker lock out
real users on purpose. Making the lockout temporary, rate limiting per IP and letting admins
unlock accounts keeps most of the protection while limiting that abuse.

### Out of scope

- TLS termination and host or network hardening (handled by the deployment, not the app)
- Password reset and email flows
- Large-scale denial of service beyond rate limiting and size limits
- Dependency and image vulnerabilities, which are covered by pip-audit and trivy in CI (phase 7)

## Findings

Issues found by the security tests and scanners (bandit, pip-audit, trivy, OWASP ZAP),
with their fix or the reason for accepting the risk.

| ID | Severity | Source | Description | Status | Notes |
|---|---|---|---|---|---|

## Reporting a vulnerability

Please **don't open a public issue**, because that discloses the bug to everyone before
it's fixed. Report it privately through GitHub instead: go to the repository's
**Security** tab and choose **Report a vulnerability**.
