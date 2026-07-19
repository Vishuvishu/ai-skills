---
name: backend-security
description: Implement, audit, and harden security in any backend application or API (Node.js/Express, Go, Python/Django/FastAPI, Java/Spring, Ruby/Rails, PHP, etc.). Use this skill whenever the user mentions security, bot attacks, spam, scraping, rate limiting, security headers, CAPTCHA/Turnstile, CORS, brute force protection, IP banning, input sanitization, SQL injection, IDOR, privilege escalation, secret/credential leaks, sensitive data exposure, session or cookie security, or asks "is my app safe?" / "is my API secure?" — even if they don't say the word "security" explicitly. Also use this skill whenever the user adds a new route, endpoint, or handler (to make sure it's secured correctly), before any production deployment or go-live, and whenever someone describes being under attack, getting scraped, seeing suspicious traffic, or wants a pre-deploy or attacker's-perspective audit of their backend.
---

# Backend Security Skill

A layered, framework-agnostic security framework for auditing, hardening, and stress-testing backend applications and APIs against bots, scrapers, brute-force attempts, data leaks, SQL/NoSQL injection, privilege escalation, and configuration mistakes.

The model is **defense in depth**: every layer below should work independently, so that a bypass of one layer doesn't compromise the whole system. The exact library or middleware differs by stack (Express vs Chi vs Django vs Spring), but the *shape* of the defense is the same everywhere — apply the layer that fits the language/framework actually in use, don't force framework-specific syntax onto a different stack.

---

## Security Architecture Overview

```
Incoming Request
     │
     ▼
[1] Path/Bot Blocking     ← Blocks known scanner patterns, bans bad IPs
     │
     ▼
[2] Security Headers      ← HSTS, CSP, X-Frame-Options, X-Content-Type-Options
     │
     ▼
[3] Rate Limiting         ← Tiered limits by route sensitivity
     │
     ▼
[4] Human Verification    ← CAPTCHA/Turnstile on public unauthenticated POST routes
     │
     ▼
[5] Input Validation      ← Sanitization, honeypot fields, schema validation
     │
     ▼
[6] Auth & Authorization  ← Authentication middleware, role checks, ownership checks
     │
     ▼
  Route Handler
     │
     ▼
[7] Output Filtering      ← Response never leaks more than the caller is owed
```

Treat this as a checklist to walk top-to-bottom for every route, not just a diagram to admire.

### Development, Deployment & Verification Layers

Beyond the runtime request pipeline, apply these layers to secure credentials, verify data privacy, harden the environment, validate business logic, and test defenses:

```
[8]  Secrets & Credentials  ← Secret Leak & Credential Security
[9]  PII & Log Security     ← Personal Data Flow & Log Exposure
[10] Env & Prod Readiness   ← Production Readiness & Configuration
[11] Logic & Access Control ← Authentication & Business Logic Security
[12] Penetration Testing    ← Penetration Testing & Vulnerability Exploitation
```

---

## Layer 1 — Path Blocking / Bot Shield

Put this at the very top of the request pipeline, before routing, auth, or business logic — a request to a forbidden path should never reach anything else.

1. **Pattern matching**: Compare the incoming path against a blocklist of known scanner/credential paths (`.env`, `.git/config`, `wp-admin`, cloud credential files, etc.). See `resources/forbidden-paths.json` for the canonical list and regex patterns — load it once at startup rather than hardcoding it inline.
2. **Auto-ban**: Any IP that hits a forbidden path is almost certainly an automated scanner, not a real user who made a typo. Blacklist that IP for the remainder of the session (in-memory, Redis, or at the edge/WAF) and reject all further requests from it by immediately returning a `404 Not Found` response, rather than a timeout, connection drop, or clear "blocked" error message. This masks the ban as if the server is simply unresponsive on those routes.
3. **Why this matters**: automated scanners hit hundreds of these paths per second looking for exposed secrets. Blocking and banning on the very first hit stops the entire scan cheaply, before it can find a real vulnerability.

---

## Layer 2 — HTTP Security Headers

Apply these to every response, regardless of framework (via middleware like Helmet in Node, `secure` in Go, Django's `SecurityMiddleware`, Spring Security headers, etc.):

| Header | Protects Against |
|---|---|
| `X-Content-Type-Options: nosniff` | MIME-sniffing attacks |
| `X-Frame-Options: DENY` | Clickjacking |
| `Strict-Transport-Security: max-age=31536000; includeSubDomains` | Protocol downgrade / cookie hijacking over HTTP |
| `Content-Security-Policy` | XSS and unauthorized script execution |

If the app serves cross-origin images, fonts, or scripts on purpose, configure `Cross-Origin-Resource-Policy` explicitly rather than disabling header protections wholesale.

---

## Layer 3 — Rate Limiting (Tiered)

Apply limits by route sensitivity, not a single global number. Use whatever the stack provides (`express-rate-limit`, `go-chi/httprate` or a token-bucket middleware, Django-ratelimit, Spring's bucket4j, an API-gateway policy, etc.):

| Tier | Routes | Limit | Purpose |
|---|---|---|---|
| **Global** | All routes | 100 req/min per IP | Prevent scraping, bulk resource exhaustion, DDoS |
| **Public Fetch** | Public data/list/search endpoints | 20 req/min per IP | Limit brute-force enumeration |
| **Auth** | `/login`, `/signup`, `/forgot-password`, OTP requests | 5 req/hour per IP (or ~5/min for login specifically) | Block automated credential stuffing and spam signups |

Auth routes are the highest-value target for attackers and must get the strictest limits — never leave them on the global tier alone.

---

## Layer 4 — Human Verification (CAPTCHA / Turnstile)

Every unauthenticated `POST` route that creates data, sends messages, or triggers a costly operation (signup, login, contact form, comment, password reset) must require a verified human token (e.g. Cloudflare Turnstile, hCaptcha, reCAPTCHA).

**Non-negotiable rules — a CAPTCHA that can be silently skipped is not a CAPTCHA:**
- If the verification secret/key is missing from the environment, **fail the request** — never fall back to "skip verification."
- If the submitted token is invalid, expired, or absent, **fail the request**.
- Never make verification optional behind a feature flag that could accidentally ship "off."

See `resources/turnstile-guide.md` for a concrete frontend + backend implementation pattern.

---

## Layer 5 — Input Validation & Honeypots

**Honeypot fields**: include a hidden field in public forms (e.g. `website_url`) that a real user would never fill in but a bot script fills automatically. If it's populated on submit, reject silently with a normal-looking success response (e.g. `200 OK`) and perform no writes — this wastes the bot's time instead of teaching it that it was caught.

**Input filtering**: validate and reject/sanitize:
- URLs where a name/bio/label is expected
- Raw HTML or script tags in fields that will ever be rendered
- Payloads containing SQL/NoSQL injection syntax (`' OR '1'='1`, `{"$ne": null}`, etc.) — this is a defense-in-depth check in addition to parameterized queries, not a replacement for them
- Oversized payloads or unexpected content types on file uploads

**Error responses**: never return stack traces, raw database errors, or internal file paths to the client. Log the detail internally; return a generic message externally.

---

## Layer 6 — Authentication & Authorization

1. **Auth middleware on every protected route** — verify there is no route that silently skips authentication because it was added quickly or copy-pasted from a public route.
2. **IDOR (Insecure Direct Object Reference) prevention** — never trust a user-, order-, or document-ID supplied by the client without checking that the *authenticated* user actually owns or is entitled to that resource. This is the single most common real-world vulnerability in CRUD APIs.
3. **Role/privilege checks happen server-side** — a role check that only hides a UI button is not a security control. Every handler that requires `admin`/`moderator`/etc. must re-verify the role from the server-side session or token, not from a client-supplied field.
4. **JWT/session hygiene** — strong, random signing secrets; expiration enforced; signature verified on every request; token blacklist or short expiry to make logout meaningful.
5. **Password reset tokens** — random, single-use, time-limited (≤15 minutes), and bound to a specific user/account.

---

## Layer 7 — Output Filtering (Personal Data & Response Shape)

No endpoint should return more than the caller is entitled to see:
- Never include password hashes, internal-only IDs, or another user's private fields in a response, even accidentally via a `SELECT *` or serializing a whole ORM object.
- Trace where personal data (email, phone, address, payment info, IP, device fingerprint) goes after collection — logs, analytics, third-party APIs, webhooks — and strip anything a downstream consumer doesn't actually need.
- Passwords must be hashed with a strong, slow hash (`bcrypt` cost ≥ 12, or `argon2`) — never stored, logged, or echoed back in plaintext, and never hashed with bare `MD5`/`SHA-256`.
- Cookies carrying session or auth data must set `HttpOnly=true`, `Secure=true` in production, and `SameSite=Lax` or `Strict`.
- Sensitive data must never live in `localStorage`/`sessionStorage` — it's readable by any script on the page, including an XSS payload.

---

## Layer 8 — Secret Leak & Credential Security

1. Move every hardcoded secret (API keys, DB connection strings, JWT/session signing secrets, OAuth client secrets, third-party API keys) into environment variables. Check config files, utility modules, router/handler setup, and code comments — secrets hide in comments more often than you'd expect.
2. Confirm `.env` (or equivalent) is in `.gitignore`, and create a checked-in `.env.example` listing every required key with a placeholder value — never a real one.
3. Confirm logs and JSON responses never print secrets, tokens, or raw connection strings, including in error handlers and unhandled-exception logging.
4. If any secret was ever committed to git history, treat it as permanently compromised — rotating it is mandatory, not optional, because history persists even after a file is deleted.

---

## Layer 9 — Personal Data Flow & Log Exposure

1. Trace every point personal data enters the system (signup forms, profile updates, payment forms, support tickets) and follow it to every place it's stored, logged, or forwarded.
2. Check every log statement and error handler for leaked PII or credentials.
3. Confirm passwords are hashed before storage, never logged, never returned in any API response.
4. Confirm cookies carrying sensitive data set the correct security flags.
5. Confirm each endpoint returns only the fields the caller needs — no excess user data, no other users' data.

---

## Layer 10 — Production Readiness & Configuration

1. **Environment validation**: the app must validate required environment variables at startup and refuse to start (with a clear error) if a critical one (DB URL, signing secret, encryption key) is missing — a silent fallback to an insecure default is worse than crashing.
2. **Debug code removal**: delete test/debug endpoints (`/debug`, `/test-db`, `/admin-backdoor`, `/seed-data`), hardcoded test credentials, and leftover debug print statements. Debug mode must default to off.
3. **Error handling policy**: standardize error responses so raw SQL, stack traces, and file paths never reach the client — mask as a generic message plus a correlation ID, with full detail going to server-side logs only.
4. **Security headers**: confirm Layer 2 headers are present on every response, not just the happy path.
5. **Rate limiting**: confirm Layer 3 tiers are actually wired up on auth routes, not just planned.
6. **CORS**: restrict allowed origins explicitly to the real frontend domain(s) in production — no wildcard `*` on any route that reads authenticated data.
7. **Database security**: TLS/SSL enforced on the DB connection in production, no default credentials, no database port exposed directly to the public internet.

---

## Layer 11 — Authentication & Business Logic Security

1. **Auth & IDOR**: every protected route has auth middleware wired in; every handler that takes a resource ID verifies the *authenticated* caller owns or is authorized for that specific resource — don't just check "is logged in," check "is logged in **as the right person**."
2. **JWT security**: expiration validated, strong random signing secret, signature verified on every request (not just decoded and trusted).
3. **Payment logic**: price, quantity, tax, and discount math must be computed independently on the server — never trust a client-submitted total. Payment-gateway webhook signatures must be verified before acting on a webhook payload.
4. **Input handling & injection**: all SQL/NoSQL queries use parameterization or an ORM's safe query builder — no string concatenation of user input into a query, ever. Sanitize any field that will later be rendered as HTML.

---

## Layer 12 — Penetration Testing & Vulnerability Exploitation
Actually try to break it, don't just read the code:
1. **ID manipulation**: increment or swap IDs in URLs, headers, and request bodies to try to read or edit another user's data.
2. **Login/role bypass**: send expired, malformed, or missing tokens; try to reach an admin/moderator-only route as a normal user by guessing the URL or editing a client-side role field.
3. **Content injection**: submit `<script>` tags, `' OR '1'='1`, and oversized/odd file uploads into every text field, query param, and upload endpoint.
4. **Internal exposure**: confirm admin panels, `.env`, `.git`, and API docs/introspection endpoints (Swagger, GraphQL introspection) aren't reachable from the public internet, or are, at minimum, authenticated.
5. **Business-logic abuse**: try negative amounts, stacking discount codes, restarting a "free trial," or referring yourself in a referral program — these are logic flaws that no amount of input sanitization catches, and they need explicit server-side rules of their own.

---

## Mandatory Checklist — Every New or Modified Route

Before any handler goes to production, confirm:

- [ ] **Auth**: does this route need authentication? Is the auth middleware actually attached (not just present elsewhere in the file)?
- [ ] **Ownership**: if this route takes a resource ID, does it verify the authenticated caller owns/can access that specific resource (IDOR check)?
- [ ] **Rate limiter**: is the correct tier (global/public/auth) applied if this route is public?
- [ ] **CAPTCHA**: if this is a public, unauthenticated `POST`, is human verification wired in and fail-closed?
- [ ] **Input validation**: are all inputs validated/sanitized, and are all DB queries parameterized?
- [ ] **Output filtering**: does the response include only fields the caller is entitled to see — no password hashes, no other users' data, no internal-only IDs?
- [ ] **Error handling**: do failures return a generic message to the client while logging full detail server-side?
- [ ] **Headers/CORS**: are security headers present, and is CORS scoped to the real frontend origin(s)?

---

## Agent Prompts & Task Runbooks

Copy-paste (and adapt) these task prompts to run a full audit pass. Each one is written so an LLM with codebase access can execute it end-to-end and report findings in a consistent, reviewable format — treat "show me X" / "report format" instructions as literal required output, not optional color.

### Task Prompt 8: Secret Leak & Credential Security
```text
Act as an application security engineer running a secret leak & credential security check. Do a full pass across the entire codebase, in every language and config format present (not just one file type).

1. Find every hardcoded secret: API keys, passwords, database connection strings (Postgres/MySQL/Mongo/Redis URIs), JWT/session signing secrets, OAuth client secrets, encryption keys, and any third-party API key (Stripe, OpenAI, SendGrid, Twilio, Firebase, AWS, Supabase, etc). Check source files, config files, infrastructure-as-code (Terraform/CloudFormation), CI/CD YAML, Dockerfiles, and code comments — secrets hide in comments and old config more often than in obvious places.
2. Apply provider-specific rules where relevant:
   - Supabase: the anon key may be used client-side ONLY if Row Level Security is enabled on every table it touches. The service role key must NEVER appear in any client-reachable code.
   - Stripe: only the publishable key may go client-side; the secret key is server-only.
   - Any OAuth client secret or JWT signing secret is server-only, with no exceptions.
3. Check frontend exposure specifically: any environment variable prefixed for client exposure (e.g. `NEXT_PUBLIC_`, `REACT_APP_`, `VITE_`, or a framework's public-config mechanism) is visible in the shipped bundle. Confirm no sensitive key uses one of these prefixes.
4. Confirm `.env`/equivalent is gitignored, and produce (or update) a `.env.example` listing every required variable name with a placeholder value — no real secrets.
5. Check every log statement, error handler, and API response body to confirm none of them print or return a secret, token, or connection string — including in stack traces on unhandled exceptions.
6. If any secret was ever hardcoded historically, flag it explicitly as compromised (git history retains it even after deletion) and state that it must be rotated immediately, not just removed going forward.

Report format: a table with columns [Secret type | File & line | Risk if leaked | Where you moved it / what you recommend]. End with a one-line overall verdict: SAFE TO DEPLOY / NOT SAFE — fix items above first.
```

### Task Prompt 9: Personal Data Flow & Log Exposure
```text
Act as a privacy/security specialist mapping personal data flow and log exposure through this app, end to end.

1. Identify every data-collection point (signup, profile edit, checkout, support form, file upload, OAuth callback, analytics beacon) and, for each, list exactly which personal fields are collected (email, phone, name, address, DOB, payment info, IP, device fingerprint, etc).
2. For each field, trace where it goes next: which database table, which third-party service (analytics, error tracking, payment processor, email/SMS provider, AI API), and whether it's included in any webhook payload sent onward.
3. Check every log statement, logger call, and error handler across the codebase — if any of them output an email, phone number, password, token, or other PII, flag it and specify the fix (redact or remove the field from the log line).
4. Confirm password handling: hashed with bcrypt/argon2/scrypt (never MD5/SHA-256 alone, never reversible encryption), never logged, never returned in any response body.
5. Check cookie and browser-storage usage: session/PII data must not sit in `localStorage`/`sessionStorage` (readable by any script including XSS); cookies carrying sensitive data need `HttpOnly`, `Secure`, and `SameSite` flags.
6. Check API response payloads for over-exposure: does any endpoint return more personal data than the calling client actually needs, or another user's data alongside the caller's own?
7. Confirm there's a way for a user to delete or anonymize their own data. If not, flag it as a gap (relevant for GDPR/CCPA-style obligations) and propose a minimal deletion/anonymization flow.

Report format: a data-flow table [Field | Collected where | Stored where | Sent to (third parties) | Logged? (Y/N, where) | Issue found | Fix applied], followed by a short summary of the biggest exposure risk.
```

### Task Prompt 10: Production Readiness & Configuration
```text
Act as a systems security engineer running a production readiness & configuration review on this application. Run every check below and report pass/fail with evidence for each — don't summarize away a failure.

1. Environment variables: does the app validate all required env vars at startup and refuse to boot with a clear error if a critical one (DB URL, signing secret, encryption key, payment secret) is missing?
2. Debug code removal: search for and flag debug-only endpoints (`/debug`, `/test-db`, `/admin-backdoor`, `/seed-data`), hardcoded test/mock credentials, commented-out code referencing incomplete security work, and leftover debug print statements. Confirm debug mode defaults to off.
3. Error handling: confirm no client-facing error response includes a stack trace, raw DB error, or file path. Detailed errors should go to server logs only, paired with a correlation ID the client can quote when reporting an issue.
4. Security headers: confirm every response sets `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Strict-Transport-Security` (1-year max-age), and a `Content-Security-Policy` scoped to the app's own domain/scripts.
5. Rate limiting: confirm auth endpoints (login, signup, password reset, OTP) are rate-limited — minimum 5 attempts/minute/IP on login, 3/hour on password reset — and that the limiter is actually mounted on the route, not just defined.
6. CORS: confirm CORS is not wildcarded (`*`) for any route that returns authenticated/private data; it should be scoped to the specific known frontend origin(s).
7. Database security: confirm the DB connection enforces TLS/SSL in production, uses no default/example credentials, and isn't reachable on an open port without authentication.

Report format: a checklist table [Check | Pass/Fail | Evidence | Fix if failed], followed by an overall GO / NO-GO recommendation.
```

### Task Prompt 11: Deep Security Audit for Authentication & Business Logic
```text
Act as an application security engineer doing a deep-dive check on this app's authentication and business logic: [payments / custom auth / complex server-side business rules — fill in what applies].

AUTHENTICATION & AUTHORIZATION:
- Check every protected route/handler for auth middleware actually being applied (not merely available).
- Check for IDOR: for every endpoint that accepts a user-, order-, document-, or resource-ID, confirm the server checks that the *authenticated caller* is the owner/authorized party for that specific ID — not just that some user is logged in.
- Check the password-reset flow specifically: tokens must be cryptographically random, single-use, expire within 15 minutes, and be bound to one specific account.
- Check JWT/session handling: strong signing secret, expiration enforced, signature verified on every request, and some mechanism (blacklist, short TTL, rotation) that makes logout actually invalidate the session.

PAYMENT LOGIC (skip if not applicable):
- Confirm price, quantity, tax, and discount totals are computed independently server-side — never trusted from the client request body.
- Try to identify whether an attacker could modify price/quantity/discount fields in a request and have the server honor them.
- Confirm webhook signatures from payment providers (Stripe, Razorpay, etc.) are verified before the webhook payload is acted on.
- Confirm access to paid features is granted only after server-side payment-status verification, not a client-reported "success" flag.

INPUT HANDLING:
- Check every form field, URL parameter, and API body field reachable by an unauthenticated or low-privilege user for SQL/NoSQL injection risk — flag any raw string concatenation into a query.
- Check for stored/reflected XSS: does any user input get rendered into HTML/templates without escaping or sanitization?
- Check file uploads: is file type validated server-side (not just by extension or client-reported MIME type), is size capped, and are uploaded files served from a location without execute permissions?

For every issue found, report: what the vulnerability is, exactly where it is in the code, a concrete example of how an attacker would exploit it, and the specific fix (with a code sketch if the fix is non-obvious).
```

### Task Prompt 12: Penetration Testing & Vulnerability Exploitation
```text
Think and act like a penetration tester actively trying to break into this app. Work through each attack path below, attempt it (or reason through exactly how you would, referencing the actual code/routes), and report what you find — don't just restate the checklist as if it were already verified.

1. Data access via ID manipulation: for every endpoint taking a user ID, order ID, or document ID, try incrementing/swapping the ID to see if another user's data is returned without an ownership check.
2. Login/session bypass: check whether any API route works with no auth token at all; check whether expired/malformed tokens are properly rejected; check for any default or leftover admin account with a known/weak credential.
3. Privilege escalation: if roles exist (user/admin/moderator/etc.), check whether a regular user can reach an admin-only route by guessing the URL, or whether a client-supplied role/claim is ever trusted instead of being re-verified server-side.
4. Feature/rate abuse: check rate limits (or their absence) on signup (mass account creation), messaging (spam), file uploads (storage exhaustion), general API calls (DDoS), and promo/referral codes (infinite reuse).
5. Content injection: attempt script tags and SQL/NoSQL injection payloads in every text field — usernames, bios, comments, search boxes, filenames — and note anywhere input isn't sanitized or queries aren't parameterized.
6. Internal exposure: check whether any of these are reachable from the public internet without auth — database admin panel, `.env` via direct URL, `.git` directory, internal-only API docs (Swagger/OpenAPI), GraphQL introspection, health-check endpoints that leak system/version info.
7. Business-logic manipulation: for payment or reward flows specifically, check whether negative amounts, infinitely stacked discounts, repeatable "free trial" resets, or self-referral are possible.

For every vulnerability found, report: what an attacker would do step by step, the realistic damage/blast radius, and the fix — prioritized with data theft and unauthorized access first, abuse/logic flaws second.
```

---

## Resources

- `resources/forbidden-paths.json` — Canonical forbidden-path list, regex patterns, and severity notes for Layer 1 path blocking.
- `resources/turnstile-guide.md` — Human-verification (CAPTCHA/Turnstile) frontend + backend implementation pattern for Layer 4.
- `resources/security-protocol.md` — Full infrastructure, session-management, observability, error-handling, and dependency-security standards that apply above and around the application layer.