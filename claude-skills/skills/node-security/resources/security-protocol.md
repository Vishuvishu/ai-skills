# Universal Security Protocol — Node.js Applications

Full standards for production-grade Node.js deployments. The SKILL.md covers the application layer in depth; this document covers infrastructure, session management, observability, and incident response.

---

## 1. Infrastructure Layer

### DNS & Proxy
- All production domains must be proxied through Cloudflare (Orange Cloud enabled)
- Cloudflare WAF must be active — blocks known malicious IPs and common injection patterns before they reach the server
- Origin IP must never be exposed publicly

### Environment & Secrets
- Credentials (DB URIs, API keys, secrets) must never be hardcoded
- `.env` files must be in `.gitignore` — verify this before every first commit on a new repo
- Production server: `chmod 600 .env`, restrict to the application user only
- Use a secrets manager (e.g., Doppler, AWS Secrets Manager) for team environments

---

## 2. Authentication & Session Management

### Cookies
All session cookies must have:
```js
{
  httpOnly: true,        // Inaccessible to client-side JS
  secure: true,          // HTTPS only (set conditionally: process.env.NODE_ENV === 'production')
  sameSite: 'lax',       // Prevents CSRF in most scenarios; use 'strict' for high-security apps
}
```

### Session Store
Use a persistent store — never rely on in-memory session storage in production:
- **Redis**: preferred for performance-sensitive apps
- **MongoDB**: fine for apps already using Mongo

In-memory stores lose all sessions on restart and can cause memory leaks under load.

### Password Hashing
- Use **Argon2** (preferred) or **Bcrypt**
- Never store plain text passwords
- Never use reversible encryption (e.g., AES) for passwords
- Minimum bcrypt cost factor: 12

---

## 3. Observability & Logging

### Structured Request Logs
Every request should log:
- `method`, `url`, `statusCode`, `ip`, `responseTimeMs`
- `userId` / `userName` for authenticated requests (enables audit trails)

### What NOT to log
- Passwords, tokens, or any credential fields
- Full request bodies on auth routes
- PII beyond what's needed for debugging

### Process Management
- Use PM2 or systemd to auto-restart on crashes
- Enable `pm2-logrotate` to prevent disk exhaustion from high-volume logs

### Incident Signals (set up alerts for these)
- Spike in 403/429 responses → active attack or scanner
- Repeated hits on forbidden paths from a single IP → targeted recon
- Auth endpoint failures exceeding normal baseline → brute force attempt

---

## 4. Error Handling

Production error handler must never leak:
- Stack traces
- Internal file paths
- Database error details
- Framework version info

```js
// Safe production error handler
app.use((err, req, res, next) => {
  console.error(err); // Log full error internally
  res.status(err.status || 500).json({
    success: false,
    msg: 'An unexpected error occurred.', // Generic to client
  });
});
```

---

## 5. Dependency Security

- Run `npm audit` regularly and on every deployment
- Keep `express`, `helmet`, and auth-related packages up to date
- Use `npm audit --production` to focus on runtime dependencies
- Consider `socket.io` and other real-time libraries as high-risk — audit carefully