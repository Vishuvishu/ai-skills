# Cloudflare Turnstile Implementation Guide

Cloudflare Turnstile is a privacy-preserving CAPTCHA replacement. This guide covers the complete implementation — Cloudflare setup, React frontend, and Node.js backend.

---

## Environment Variables

Turnstile uses two separate keys that go in two different places:

| Key | Where it goes | Exposed to users? |
|---|---|---|
| **Site Key** (public) | Frontend `.env` | Yes — embedded in the page |
| **Secret Key** (private) | Backend `.env` | Never — server only |

They must always come from their respective `.env` files. There is no fallback. If a key is missing, it is a configuration error that must be fixed, not bypassed.

**Frontend `.env` — local development:**
```env
# Cloudflare's official test Site Key — passes verification but shows no visible widget
VITE_TURNSTILE_SITE_KEY=1x00000000000000000000AA
```

**Frontend `.env` — production:**
```env
# Replace with your real Site Key from Cloudflare Dashboard
VITE_TURNSTILE_SITE_KEY=your_real_site_key_here
```

**Backend `.env` — local development:**
```env
# Cloudflare's official test Secret Key — works with the test Site Key above
TURNSTILE_SECRET_KEY=1x0000000000000000000000000000000AA
```

**Backend `.env` — production:**
```env
# Replace with your real Secret Key from Cloudflare Dashboard
TURNSTILE_SECRET_KEY=your_real_secret_key_here
```

> Never put `TURNSTILE_SECRET_KEY` in the frontend and never put `VITE_TURNSTILE_SITE_KEY` in the backend. Keep them separate and let the app crash loudly if either is missing — that's intentional.

---

## 1. Cloudflare Setup

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Navigate to **Turnstile** → **Add Site**
3. Choose a widget type:
   - **Managed** *(recommended)*: Cloudflare decides when to show a challenge
   - **Non-interactive**: Shows a loading spinner, no challenge
   - **Invisible**: No widget shown unless the user looks suspicious
4. Copy your **Site Key** (public) and **Secret Key** (private) — add them to your production `.env` as shown above

---

## 2. Frontend (React)

The frontend renders the widget and captures the token. The token is then sent to your backend with the form submission — the backend does the actual verification.

```bash
npm install @marsidev/react-turnstile
```

```tsx
import { Turnstile } from '@marsidev/react-turnstile';
import { useState } from 'react';

const SignupForm = () => {
  const [token, setToken] = useState('');

  const handleSubmit = async () => {
    if (!token) return; // Widget hasn't completed yet
    await fetch('/api/signup', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ captchaToken: token, ...formData }),
    });
  };

  return (
    <div>
      {/* Your form inputs */}

      <Turnstile
        siteKey={import.meta.env.VITE_TURNSTILE_SITE_KEY}
        onSuccess={(token) => setToken(token)}
        onError={() => setToken('')}      // Widget failed — block submission
        onExpire={() => setToken('')}     // Token expired — block submission
      />

      <button onClick={handleSubmit} disabled={!token}>
        Submit
      </button>
    </div>
  );
};
```

> The frontend widget is UX only — it does not verify anything. A bot can skip it entirely and POST directly to your API. The backend verification in the next section is what actually enforces it.

---

## 3. Backend (Node.js)

The backend sends the token to Cloudflare's API to verify it was issued legitimately. This is the real enforcement layer.

### Utility Function

```js
// utils/verifyTurnstile.js
import fetch from 'node-fetch';

export const verifyTurnstile = async (token) => {
  const secretKey = process.env.TURNSTILE_SECRET_KEY;

  // Missing key = broken config. Crash loudly, never fall back to test keys.
  if (!secretKey) {
    throw new Error('TURNSTILE_SECRET_KEY is not set in environment variables');
  }

  if (!token) return false;

  const response = await fetch(
    'https://challenges.cloudflare.com/turnstile/v0/siteverify',
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
      body: `secret=${secretKey}&response=${token}`,
    }
  );

  const data = await response.json();
  return data.success === true;
};
```

### Middleware (reusable across routes)

```js
// middleware/requireTurnstile.js
import { verifyTurnstile } from '../utils/verifyTurnstile.js';

export const requireTurnstile = async (req, res, next) => {
  const token = req.body?.captchaToken;
  const isHuman = await verifyTurnstile(token);
  if (!isHuman) {
    return res.status(403).json({ success: false, msg: 'Security verification failed.' });
  }
  next();
};
```

### Route Usage

```js
import { requireTurnstile } from '../middleware/requireTurnstile.js';

router.post('/signup', requireTurnstile, async (req, res) => {
  const { captchaToken, ...userData } = req.body;
  // Token already verified — proceed safely
});

router.post('/login', requireTurnstile, async (req, res) => {
  // ...
});
```

---

## Security Rules

- **Always fail closed**: if the secret key is missing or Cloudflare's API is unreachable, throw — never silently pass
- **Never fall back to test keys**: if `TURNSTILE_SECRET_KEY` is missing, it is a deployment error — fix the env, don't patch the code
- **Server-side only**: never expose the Secret Key to the frontend or any client-side code
- **No bypass flags**: never add `SKIP_CAPTCHA=true` or similar — it will eventually be left on in production
- **Token is single-use**: Cloudflare invalidates tokens after one verification — never cache or reuse them