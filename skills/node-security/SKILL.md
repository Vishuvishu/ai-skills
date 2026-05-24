---
name: node-security
description: Implement, audit, and harden security in Node.js/Express applications. Use this skill whenever the user mentions security, bot attacks, spam, rate limiting, headers, CAPTCHA, Turnstile, helmet, CORS, brute force protection, IP banning, input sanitization, honeypots, or asks "is my app safe?" — even if they don't say "security" explicitly. Also trigger when the user adds a new route or endpoint to make sure it's secured correctly. If someone describes being under attack, getting scraped, or noticing suspicious traffic, use this skill immediately.
---

# Node.js Security Skill

This skill provides a layered security framework for hardening Node.js/Express applications against bot attacks, automated scanners, brute force attempts, and unauthorized data access.

The security model follows **defense in depth**: multiple independent layers so that bypassing one doesn't compromise the whole system.

---

## Security Architecture Overview

```
Incoming Request
     │
     ▼
[1] BotShield        ← Blocks known scanner patterns, bans bad IPs
     │
     ▼
[2] Helmet Headers   ← Sets HTTP security headers
     │
     ▼
[3] Rate Limiters    ← Tiered limits by route sensitivity
     │
     ▼
[4] Turnstile CAPTCHA ← Verifies human on public POST routes
     │
     ▼
[5] Input Validation ← Sanitizes data, rejects honeypot triggers
     │
     ▼
[6] Auth Middleware  ← isAuthenticated, role checks
     │
     ▼
  Route Handler
```

---

## Layer 1 — BotShield (First Line of Defense)

Place this middleware **at the very top** of `index.js`, before anything else.

```js
// index.js
app.use(botShield); // MUST be first
app.use(helmet());
// ... rest of middleware
```

BotShield does two things:
- **Pattern matching**: Checks the request path against `resources/forbidden-paths.json` (env files, git config, CMS scanners, etc.)
- **Auto-ban**: Any IP hitting a forbidden path is immediately blacklisted for the session (in-memory or Redis). All subsequent requests from that IP are rejected without processing.

For the forbidden path list and regex patterns, see `resources/forbidden-paths.json`.

---

## Layer 2 — HTTP Security Headers (Helmet)

Use `helmet()` globally. Key headers it sets:

| Header | Protects Against |
|---|---|
| `X-Content-Type-Options: nosniff` | MIME sniffing attacks |
| `X-Frame-Options: DENY` | Clickjacking |
| `Content-Security-Policy` | XSS, unauthorized script execution |

If serving cross-origin images or scripts, configure `Cross-Origin-Resource-Policy` explicitly — don't disable it entirely.

---

## Layer 3 — Rate Limiting (Tiered)

Use `express-rate-limit`. Apply limits based on route sensitivity:

| Tier | Routes | Limit | Purpose |
|---|---|---|---|
| Global | All routes | 100 req/min | Prevent scraping and DDoS |
| Public fetch | QR code, public data endpoints | 20 req/min | Limit enumeration |
| Auth | `/login`, `/signup`, `/forgot-password` | 5 req/hr | Stop brute force |

Auth routes get the strictest limits — these are the highest-value targets.

---

## Layer 4 — Human Verification (Cloudflare Turnstile)

All unauthenticated `POST` routes that create or access data **must** require a valid Turnstile token.

**Critical rules:**
- If `TURNSTILE_SECRET_KEY` is missing from env → **fail the request** (don't skip)
- If the token is invalid or absent → **fail the request**
- Never make verification optional or conditional on an env flag

For the full frontend + backend implementation, see `resources/turnstile-guide.md`.

---

## Layer 5 — Input Validation & Honeypots

**Honeypot fields**: Include a hidden `website_url` field (or similar) in forms. Bots fill every field — if it's populated, reject silently with a 200 response (don't tell bots they were caught).

**Input filtering**: Reject inputs that contain:
- URLs in name/bio/label fields
- HTML or script tags anywhere unexpected
- Excessive emojis in fields where they don't belong

**Error responses**: Never return stack traces, internal paths, or DB error details. Log internally, return generic messages to the client.

---

## Mandatory Checklist — New or Modified Routes

Before any route goes to production:

- [ ] Authentication required? Add `isAuthenticated` middleware
- [ ] Rate limiter applied? Choose the right tier
- [ ] Public POST? Turnstile verification added
- [ ] Sensitive data returned? Validate and authorize the requesting user's access to those IDs
- [ ] Error messages generic? No stack traces or internal paths leaked

---

## Infrastructure (Outer Layer)

These aren't enforced in code but are part of the security posture:

- **Cloudflare proxy** on all production domains — hides origin IP
- **WAF** active — blocks known malicious IPs and injection patterns at the edge
- **`.env` files** never committed; `chmod 600` on production server
- **Cookies**: `HttpOnly`, `Secure` (prod only), `SameSite: Lax`
- **Password hashing**: Argon2 or Bcrypt — never plain text or reversible encryption
- **Session store**: Redis or MongoDB, not in-memory

For the full infrastructure and observability protocol, see `resources/security-protocol.md`.

---

## Resources

- `resources/forbidden-paths.json` — Path patterns and regexes for BotShield
- `resources/turnstile-guide.md` — Cloudflare Turnstile frontend + backend implementation
- `resources/security-protocol.md` — Full infrastructure, session, logging, and incident response standards