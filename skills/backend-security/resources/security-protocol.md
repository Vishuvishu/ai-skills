# Universal Security Protocol — Backend Applications

Full standards for production-grade backend deployments, in any language or framework. The main `SKILL.md` covers the application layer in depth; this document covers infrastructure, session management, observability, error handling, and dependency security — the layer above and around the application code.

---

## 1. Infrastructure Layer

### DNS & Proxy
- All production domains should sit behind a proxy/CDN with a WAF (e.g. Cloudflare with the orange cloud enabled, or an equivalent at your cloud provider) so known malicious IPs and common injection patterns are blocked before they ever reach the origin server.
- The origin server's IP must never be exposed publicly — if it leaks, the WAF and proxy layer can be bypassed entirely.

### Environment & Secrets
- Credentials (DB URIs, API keys, signing secrets) must never be hardcoded, in any language — this is true whether the app is Node, Go, Python, Java, or PHP.
- `.env`/equivalent config files must be in `.gitignore` — verify this before the very first commit on a new repo, since a secret committed once is compromised forever (git history retains it).
- On the production server, restrict permissions on env/secret files to the application user only (e.g. `chmod 600 .env`).
- For team environments, prefer a secrets manager (Doppler, AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) over files passed around manually.

---

## 2. Authentication & Session Management

### Cookies
Every session or auth cookie must set:
- `HttpOnly` — inaccessible to client-side JavaScript, so an XSS payload can't read it.
- `Secure` — HTTPS only, at minimum in production (`NODE_ENV === 'production'`, `DEBUG=False`, etc. depending on stack).
- `SameSite=Lax` — prevents CSRF in most scenarios; use `Strict` for high-security apps like banking or admin panels.

### Session Store
Use a persistent, shared store — never rely on in-memory session storage in production, regardless of framework:
- **Redis** — preferred for performance-sensitive apps.
- **A relational or document database** the app already uses — fine if performance isn't the bottleneck.

In-memory stores lose every session on restart or redeploy, and can leak memory under sustained load.

### Password Hashing
- Use **Argon2** (preferred) or **Bcrypt** — never a fast general-purpose hash like MD5 or bare SHA-256, and never reversible encryption.
- Minimum bcrypt cost factor: 12 (raise as hardware gets faster).
- Never store, log, or return plaintext passwords under any circumstance, including "just for debugging."

---

## 3. Observability & Logging

### Structured Request Logs
Every request should log at minimum:
- `method`, `path`, `statusCode`, `sourceIp`, `responseTimeMs`
- `userId` (or equivalent) for authenticated requests, to enable audit trails when investigating an incident

### What NOT to log
- Passwords, tokens, session IDs, or any other credential material
- Full request bodies on auth routes (they typically contain passwords or tokens)
- PII beyond what's actually needed for debugging the specific issue at hand

### Process Management
- Use a process supervisor appropriate to the stack (PM2 or systemd for Node, systemd or a container orchestrator for Go/Python/Java) so the app restarts automatically on crash.
- Enable log rotation (e.g. `pm2-logrotate`, `logrotate`, or the platform's built-in log-shipping) to avoid disk exhaustion from high-volume logs.

### Incident Signals — set up alerts for these
- A spike in `403`/`429` responses → active attack or scanner activity.
- Repeated hits on forbidden paths (see `forbidden-paths.json`) from a single IP → targeted reconnaissance.
- Auth-endpoint failure rate exceeding the normal baseline → brute-force or credential-stuffing attempt in progress.

---

## 4. Error Handling

A production error handler must never leak, to the client, in any framework:
- Stack traces
- Internal file paths
- Raw database error messages or constraint names
- Framework/library version info

```text
// Conceptual shape of a safe production error handler, regardless of language:
on unhandled error:
    log(err, fullDetail=True)  // full detail goes to server-side logs only
    respond(
        status = err.status or 500,
        body = { success: false, message: "An unexpected error occurred.", correlationId: <id> }
    )
```

Pair the generic client-facing message with a correlation ID so a user or support agent can reference the exact incident in the server logs without exposing anything sensitive in the response itself.

---

## 5. Dependency Security

- Run the stack's dependency-audit tool regularly and on every deployment (`npm audit`, `pip-audit`, `govulncheck`, `bundler-audit`, `mvn dependency-check`, etc.).
- Keep security-relevant packages (web framework, security-headers middleware, auth libraries) up to date — these are the packages attackers target first when a CVE drops.
- Scope audits to production/runtime dependencies where the tool supports it, to focus attention on what's actually deployed.
- Treat real-time or file-upload libraries (WebSocket servers, `socket.io`-style libraries, multipart parsers) as higher risk than average — audit these more carefully, since they parse untrusted input by design.