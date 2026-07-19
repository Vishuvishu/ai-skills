# Human Verification Implementation Guide (CAPTCHA / Turnstile)

Concrete pattern for wiring a human-verification check (Cloudflare Turnstile, hCaptcha, or reCAPTCHA — the flow is identical across providers) into any backend stack. Apply this to every unauthenticated `POST` route that creates data, sends a message, or triggers a costly operation.

---

## The flow

1. **Frontend**: render the provider's widget on the form. On submit, the widget produces a short-lived token alongside the rest of the form data.
2. **Backend**: before doing anything else with the request (before touching the database, before sending an email), take that token and verify it server-side by calling the provider's verification endpoint with the token and your **secret key**.
3. **Only if verification succeeds**, proceed to process the request normally.

The verification call itself is a simple server-to-server POST — no client library is needed on the backend, just an HTTP request.

---

## Backend verification (conceptual, applies to any language)

```text
function verifyHumanToken(token, secretKey):
    if secretKey is missing or empty:
        # Fail closed — do not silently allow the request through
        raise ConfigurationError("Human verification secret is not configured")

    if token is missing or empty:
        return false

    response = POST to provider's verification URL with:
        secret = secretKey
        response = token
        remoteip = requester's IP (optional but recommended)

    return response.success == true
```

Wire this as middleware/decorator on the specific routes that need it, not globally — most of an API (authenticated routes, internal routes, read-only public routes) doesn't need human verification.

```text
handler for POST /signup:
    if not verifyHumanToken(request.body.turnstileToken, env.TURNSTILE_SECRET_KEY):
        return 400, { error: "Human verification failed" }
    # ... proceed with signup logic
```

---

## Non-negotiable rules

- **Fail closed on missing config.** If the secret key isn't set in the environment, the request must fail — never treat a missing secret as "verification not required."
- **Fail closed on invalid/missing token.** A request with no token, an expired token, or a token that fails provider verification must be rejected — never processed "just this once."
- **No environment-flag bypass.** Don't gate verification behind a flag like `SKIP_CAPTCHA=true` that could accidentally ship to production, or that a developer could flip during debugging and forget to flip back.
- **Verify server-side, always.** Client-side widget completion alone proves nothing — the token must be checked against the provider's server on your backend before it's trusted.
- **Scope narrowly.** Apply this to public, unauthenticated, state-changing routes (signup, login, contact form, comment/review submission, password reset request). Authenticated routes should rely on session/auth checks instead, not repeated CAPTCHAs.

---

## Choosing where to apply it

| Route type | Needs human verification? |
|---|---|
| Public signup / login | Yes |
| Public contact / feedback form | Yes |
| Password reset request | Yes |
| Comment/review submission by anonymous users | Yes |
| Authenticated API calls from a logged-in user | No — rely on auth + rate limiting instead |
| Internal/admin routes | No — rely on auth + IP allowlisting instead |
| Read-only public GET endpoints | No — rate limiting is the right control here, not CAPTCHA |