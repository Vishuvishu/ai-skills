---
name: vercel-nextjs
description: >
  Comprehensive guide for deploying and optimizing Next.js projects on Vercel.
  Use this skill whenever the user mentions deploying on Vercel, asks about
  vercel.json configuration, wants to optimize their Vercel deployment, mentions
  slow Vercel performance, asks about Vercel regions, functions, cron jobs,
  image optimization, headers, redirects, rewrites, or any Vercel-related
  configuration. Trigger even if they just say "I deploy on Vercel" or "I use
  Vercel" — always provide an optimized vercel.json and relevant advice.
---

# Vercel Next.js Deployment Skill

## Goal
Give the user a fully optimized `vercel.json` for their Next.js project and explain every relevant option clearly. Always tailor advice based on what the user shares about their stack (DB, auth, region, use case).

---

## Step 1 — Always Start With Schema Line

Every `vercel.json` should start with:
```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json"
}
```
This enables autocomplete + validation in VS Code. Always include it.

---

## Step 2 — Region (Most Impactful for Non-US Users)

Default region is `iad1` (US East). This causes high latency for non-US users.

**Ask or infer** where the user's primary audience is, then set accordingly:

| Region Code | Location |
|---|---|
| `bom1` | Mumbai, India 🇮🇳 |
| `sin1` | Singapore 🇸🇬 |
| `hnd1` | Tokyo, Japan 🇯🇵 |
| `cdg1` | Paris, France 🇫🇷 |
| `lhr1` | London, UK 🇬🇧 |
| `sfo1` | San Francisco, US 🇺🇸 |
| `iad1` | Washington DC, US (default) 🇺🇸 |
| `gru1` | São Paulo, Brazil 🇧🇷 |

```json
{
  "regions": ["bom1"]
}
```

- Hobby plan: 1 region only
- Pro plan: multiple regions allowed
- Static assets are always on the global CDN regardless of region — only API/server functions are affected

**Tip for Indian users**: `bom1` (Mumbai) cuts API latency from ~300ms to ~60ms compared to `iad1`.

---

## Step 3 — Fluid Compute

Always enable — reduces cold starts by handling concurrency smartly:

```json
{
  "fluid": true
}
```

> Note: Enabled by default on new projects created after April 2025. Still safe to explicitly set.

---

## Step 4 — Functions Configuration

Configure per-route function behavior using glob patterns:

```json
{
  "functions": {
    "api/*.js": {
      "maxDuration": 30
    },
    "api/heavy-task.js": {
      "maxDuration": 60,
      "supportsCancellation": true
    }
  }
}
```

**maxDuration limits by plan:**
- Hobby: max 60 seconds
- Pro: max 300 seconds (5 min)
- Enterprise: max 900 seconds (15 min)

**Per-function region** (Pro/Enterprise — useful if different API routes access different DBs):
```json
{
  "functions": {
    "api/india-data.js": {
      "regions": ["bom1"],
      "functionFailoverRegions": ["sin1"]
    }
  }
}
```

---

## Step 5 — Security Headers

Always add security headers. Best practice for any production app:

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-XSS-Protection", "value": "1; mode=block" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" }
      ]
    },
    {
      "source": "/service-worker.js",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=0, must-revalidate" }
      ]
    }
  ]
}
```

---

## Step 6 — Image Optimization

If using `next/image` with external image sources, configure allowed domains:

```json
{
  "images": {
    "sizes": [320, 640, 960, 1280, 1920],
    "formats": ["image/webp", "image/avif"],
    "minimumCacheTTL": 3600,
    "remotePatterns": [
      {
        "protocol": "https",
        "hostname": "your-storage.s3.amazonaws.com"
      }
    ],
    "dangerouslyAllowSVG": false
  }
}
```

- `avif` gives better compression than `webp` but slower to encode
- `minimumCacheTTL` in seconds — 3600 = 1 hour, 86400 = 1 day

---

## Step 7 — Redirects & Rewrites

**Redirects** (browser URL changes):
```json
{
  "redirects": [
    { "source": "/old-page", "destination": "/new-page", "permanent": true },
    { "source": "/blog/:slug", "destination": "/posts/:slug", "permanent": false }
  ]
}
```
- `permanent: true` → 308 redirect (cached by browser)
- `permanent: false` → 307 redirect (not cached)

**Rewrites** (URL stays same, content changes — useful for proxying):
```json
{
  "rewrites": [
    { "source": "/api/proxy/:path*", "destination": "https://external-api.com/:path*" }
  ]
}
```

---

## Step 8 — Cron Jobs

Run scheduled tasks (calls your API route on a schedule):

```json
{
  "crons": [
    { "path": "/api/sync-data", "schedule": "0 0 * * *" },
    { "path": "/api/cleanup", "schedule": "0 2 * * 0" }
  ]
}
```

Common schedule patterns:
- `* * * * *` — every minute
- `0 * * * *` — every hour
- `0 0 * * *` — every day at midnight
- `0 9 * * 1-5` — weekdays at 9am

> Crons only run on **production** deployments.

---

## Step 9 — Build Settings

Override defaults if needed:

```json
{
  "buildCommand": "npm run build",
  "installCommand": "npm install",
  "outputDirectory": ".next",
  "framework": "nextjs",
  "ignoreCommand": "git diff --quiet HEAD^ HEAD ./"
}
```

- `ignoreCommand`: exits 0 = skip build, exits 1 = run build. Useful to skip rebuilds when only docs changed.
- `bunVersion: "1.x"` — use Bun instead of Node.js for functions (experimental)

---

## Step 10 — URL Hygiene

```json
{
  "cleanUrls": true,
  "trailingSlash": false
}
```

- `cleanUrls`: removes `.html` extensions — `/about.html` becomes `/about`
- `trailingSlash: false`: `/about/` redirects to `/about`

---

## Complete Optimized Template (Next.js, Indian Audience)

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "regions": ["bom1"],
  "fluid": true,
  "framework": "nextjs",
  "cleanUrls": true,
  "trailingSlash": false,
  "functions": {
    "api/*.js": {
      "maxDuration": 30
    }
  },
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-XSS-Protection", "value": "1; mode=block" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
      ]
    }
  ],
  "images": {
    "sizes": [320, 640, 960, 1280, 1920],
    "formats": ["image/webp", "image/avif"],
    "minimumCacheTTL": 3600
  }
}
```

---

## How to Verify the Config Worked

After deploying, check the region is correct:

```bash
curl -I https://your-project.vercel.app/api/any-route
```

Look for: `x-vercel-id: bom1::...` — the prefix confirms the active region.

---

## Common Performance Issues (diagnose these too)

| Symptom | Likely Cause | Fix |
|---|---|---|
| API routes slow globally | Wrong region (iad1 default) | Set `regions: ["bom1"]` or nearest |
| First request slow, then fast | Cold start | Enable `fluid: true` |
| DB calls slow on Vercel | New connection per request (serverless) | Use PgBouncer / Neon / Prisma Accelerate |
| All pages slow | Everything is SSR, no caching | Use ISR (`revalidate`) or static generation |
| Images slow | No optimization config | Set `images.formats` and `minimumCacheTTL` |

---

## Reference

Full official docs: https://vercel.com/docs/project-configuration/vercel-json  
All region codes: https://vercel.com/docs/regions