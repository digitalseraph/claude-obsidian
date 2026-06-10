---
type: concept
title: "Cloudflare Access Breaks Session-Based PWA Logins"
created: 2026-06-10
updated: 2026-06-10
tags:
  - cloudflare-access
  - zero-trust
  - better-auth
  - oauth
  - pwa
  - workers
  - debugging
  - marietta-na
status: developing
related:
  - "[[PDF Bank-Statement Parsing on Workers]]"
---

# Cloudflare Access Breaks Session-Based PWA Logins

A self-hosted app that has its own login (better-auth OAuth) **and** is also placed behind Cloudflare Access (Zero Trust) for pre-launch privacy will fail to log users in, in ways that look like application bugs but are actually edge interception. The failure mode is amplified by a PWA service worker that serves the cached app shell offline, which hides the Access redirect and makes a working session look logged-out.

This is the reusable lesson from fixing Google login on `mariettana.digitalseraph.com` (the [[Marietta NA]] project: a Cloudflare Worker running Hono + better-auth, serving a React 19 + Vite PWA, single origin in production).

## The core conflict

Two independent auth layers stack on the same hostname:

1. **Cloudflare Access** (outer) — gates the whole host so the public can't see the pre-launch site. Enforced at Cloudflare's edge; an unauthenticated request gets a `302` to `<team>.cloudflareaccess.com/cdn-cgi/access/login/...` and a `www-authenticate: Cloudflare-Access` header. The request never reaches the Worker.
2. **better-auth** (inner) — the app's own user accounts (Google/Zoho/email), with sessions in a cookie.

When Access sits in front of the *paths the inner login needs*, the outer layer intercepts the inner layer's traffic. The OAuth round-trip and the session-check XHR both bounce off Access and return the Access login HTML instead of the JSON the SPA expects.

**The PWA makes it invisible.** The service worker renders the cached login/app shell with no network call, so the user sees a normal page. Only the live `fetch()` calls hit the network, bounce off Access, and fail silently. A valid session looks logged-out because the app can never read it.

## The three root causes (and how each was found)

The decisive diagnostic was always the same: **`curl` the production endpoint and read the edge response.** A `302 -> cloudflareaccess.com` means Access ate the request before the Worker; a Worker JSON/401 means the request got through. The position of the failure (before vs after the redirect to the IdP) tells you which layer is broken.

1. **Access gated `/api/auth/*`.** Clicking "Continue with Google" called `POST /api/auth/sign-in/social`, which 302'd to Access instead of returning `{ url: "accounts.google.com/..." }`. The frontend saw no `url` and showed "Google sign-in failed" — *before any redirect to Google*, which ruled out a Google-console `redirect_uri` problem.
   **Fix:** a more-specific Access application scoped to the auth path with a `bypass` / `include: everyone` policy. Access matches the most-specific path, so the auth path opens while the catch-all keeps gating everything else.

2. **No `prompt` on the Google provider.** Once the edge was open, Google never showed the account chooser — it silently reused whatever Google session was already active in the browser (a personal `@gmail.com`), which isn't in the allowed Workspace domain, so better-auth's invite-gate hook threw and redirected to `/?error=invite_code_required`. The intended service account (`web@mariettana.org`) could never be selected.
   **Fix:** set `prompt: "select_account"` on the better-auth Google `socialProviders` config to force the chooser. (`better-auth` forwards `options.prompt` into the authorization URL; valid values include `select_account`, `consent`, `select_account consent`.) Shipped in `api/src/auth/better-auth.ts`, commit `ca48d18`.

3. **Access still gated `/api/v1/*`.** Login now completed and set the session cookie, but the app still showed "Sign in." The signed-in check is `GET /api/v1/me`, which was *not* under the bypassed auth path, so it 302'd to Access. The SPA (booted from PWA cache, with no live Access session) read a non-200 and treated the user as anonymous. The session existed; the app couldn't read it.
   **Fix:** widen the bypass application's path to cover the whole API (`/api*`). The app's own better-auth session + role middleware protect `/me` and `/admin`; public endpoints are public by design — so Access only needs to gate the HTML shell.

## The general rule

**A session-based SPA/PWA behind Cloudflare Access does not compose unless the API path is bypassed.** Put Access in front of the HTML/app-shell only; let the application's own auth protect the API. Otherwise every authenticated XHR depends on the browser carrying a live Access JWT, and the moment that session lapses (or the PWA shell renders without one), the app silently appears logged-out while the real session is fine.

If the host is meant to become fully public at launch, the cleaner end state is to delete the catch-all Access application entirely rather than maintain path carve-outs.

## Operational gotchas

- **Most-specific path wins.** A `bypass` app on `host/api*` overrides a catch-all `deny` app on `host`. This is the carve-out mechanism.
- **Updating an Access app via API uses `PUT`** (full-object replace), not `PATCH` — `PATCH` returns error `10405: Method not allowed for this authentication scheme`. `GET` the app, change the path fields, `PUT` it back. A `PUT` to the app does not drop its attached policies.
- **The Cloudflare MCP OAuth token is read-only for Access** — it can list apps but writes return `10000: Authentication error`. Creating/editing Access apps needs a scoped API token (`Account > Access: Apps and Policies > Edit`).
- **Don't pull production user PII to diagnose** — the failure is visible at the edge with anonymous `curl`; the user table is never needed.

This is the same family of lesson as [[PDF Bank-Statement Parsing on Workers]]: on Cloudflare Workers, reproduce against the real edge/runtime with the smallest possible probe, and read the layer boundaries before touching application code.
