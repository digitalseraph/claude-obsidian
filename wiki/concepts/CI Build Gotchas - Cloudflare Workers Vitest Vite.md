---
type: concept
title: "CI & Build Gotchas: Cloudflare Workers + Vitest + Vite"
created: 2026-05-17
updated: 2026-05-17
tags:
  - ci
  - cloudflare-workers
  - vitest
  - vite
  - wrangler
  - gotchas
status: current
related:
  - "[[LLM Wiki Pattern]]"
sources:
  - "marietta-na CI repair + e2e/lighthouse harness (PRs #5–#10)"
---

# CI & Build Gotchas: Cloudflare Workers + Vitest + Vite

Hard-won, reusable failure modes from repairing a fully-red CI on a
Cloudflare Workers (Hono + Drizzle) API + Vite/React SPA monorepo.

## 1. `@vitest/coverage-v8` is incompatible with `@cloudflare/vitest-pool-workers`

`@vitest/coverage-v8` imports `node:inspector/promises`, which the
**workerd** runtime does not provide. Running `vitest run --coverage`
in the Workers pool fails with `No such module "node:inspector/promises"`
→ "no tests" + nonzero exit.

- **Fix:** do not pass `--coverage` for the Worker package. Run plain
  `vitest run` there. v8 coverage works fine for the jsdom (frontend)
  package — install `@vitest/coverage-v8` matching the vitest major.

## 2. `wrangler@4` requires Node ≥ 22

Any CI job that runs `wrangler dev`/`deploy` (directly or via a server
for e2e/lighthouse) dies on Node 20: *"Wrangler requires at least
Node.js v22.0.0"* → server never starts.

- **Fix:** `node-version: 22` (or higher) in all CI jobs touching wrangler.

## 3. CI server pattern for e2e / Lighthouse of a Worker-served SPA

The app is served in prod by the Worker (static `dist` via `assets`
binding + SPA fallback + `/api`). For faithful e2e/Lighthouse:

- Build the frontend, then **migrate + seed the local D1** (`wrangler
  d1 migrations apply … --local`, then seed SQL).
- Start the **api worker** (`wrangler dev`) as the test server — it
  serves the SPA *and* the API on one origin (mirrors prod; no
  static-only API 404 console errors).
- Neutralize external deps: `wrangler dev --var BMLT_ROOT_URL:`
  (empty var) so a meetings page makes no external call and renders
  cleanly.
- Raise readiness timeouts: Playwright `webServer.timeout` and lhci
  `startServerReadyTimeout` (~120 s) + lhci `startServerReadyPattern`
  matching wrangler's `Ready on …`. Cold-start is slow.
- Keep the server command **build-free** (do build/migrate/seed as
  separate CI steps) so the readiness window isn't blown by a rebuild.

## 4. ESLint `jsx-a11y/label-has-associated-control` + custom inputs

`<label><span>…</span><CustomInput/></label>` is correctly associated,
but the rule can't see through a non-native component → false-positive
flood (here: 30 of 32). **Don't rewrite the markup.**

- **Fix:** rule option `controlComponents: ["Input","Select","Textarea"]`
  (the design-system components). Not a weakening — it informs the rule
  about custom controls. Keep severity at error.

## 5. Vite `?raw` text imports inline into the JS bundle

`import txt from "./big.txt?raw"` embeds the file as a JS string in the
entry chunk. A 190 KB policy `.txt` blew the bundle budget.

- **Fix:** move the file to `public/` and `fetch()` it at runtime
  (lazy fallback). Removes it from JS entirely. Splitting (`manualChunks`)
  only fixes per-chunk limits, never the *total* JS budget.

## 6. Two silent foot-guns

- **`git add` with any nonexistent pathspec aborts the whole add.**
  `git add a b c missing/` stages *nothing* and the subsequent commit
  silently omits a, b, c. Stage only paths that exist; verify with
  `git status` before commit.
- **A stale `node_modules/.vite` cache masks removed-import breakage.**
  A local build can pass while CI (clean `npm ci`, no cache) fails on
  the same commit. Always clean-verify (`rm -rf node_modules/.vite
  dist`) to mirror CI before trusting a green local build.
