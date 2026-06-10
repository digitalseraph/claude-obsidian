---
type: meta
title: "Hot Cache"
updated: 2026-06-10
tags:
  - meta
  - hot-cache
status: evergreen
related:
  - "[[index]]"
  - "[[log]]"
  - "[[Wiki Map]]"
  - "[[getting-started]]"
  - "[[DragonScale Memory]]"
---

# Recent Context

Navigation: [[index]] | [[log]] | [[overview]]

## Last Updated

2026-06-10: Cross-project save from marietta-na — new concept page [[Cloudflare Access Breaks Session-Based PWA Logins]]. Google login on the pre-launch host `mariettana.digitalseraph.com` failed because the host sits behind Cloudflare Access (Zero Trust) AND the app has its own better-auth OAuth login; the two layers collide. Three root causes, each found by `curl`-ing the prod endpoint and reading whether it 302s to `cloudflareaccess.com` (edge) or returns a Worker response: (1) Access gated `/api/auth/*` so `POST /api/auth/sign-in/social` bounced before any Google redirect → "Google sign-in failed"; fixed with a more-specific bypass Access app. (2) better-auth Google provider had no `prompt`, so Google silently reused the browser's active gmail session → invite-gate threw `?error=invite_code_required`; fixed with `prompt: "select_account"` (commit `ca48d18`). (3) Access still gated `/api/v1/*`, so the post-login session check `GET /api/v1/me` bounced and the SPA (booted from PWA cache, no live Access JWT) showed "Sign in" despite a valid session; fixed by widening the bypass to `/api*`. General lesson: a session-based SPA/PWA behind Cloudflare Access does not compose unless the API path is bypassed — gate the HTML shell with Access, let the app's own auth protect the API. Gotchas: Access app updates use PUT not PATCH (PATCH → 10405); the Cloudflare MCP token is read-only for Access (writes → 10000); most-specific path wins for carve-outs.

2026-05-26 (later): Third closeout — hidden source maps + Sentry uploader stub. `vite.config.ts` `build.sourcemap: "hidden"` so `.map` files exist on disk but bundles carry no `sourceMappingURL` pragma (browsers don't auto-fetch). `frontend/scripts/upload-sourcemaps.mjs` shells out to `@sentry/cli@^2 releases files <sha> upload-sourcemaps dist --url-prefix "~/assets" --validate` when `SENTRY_AUTH_TOKEN`/`ORG`/`PROJECT` are all set; logs `skipped` + exits 0 otherwise so the deploy pipeline can chain it from day one. Either branch then strips `.map` from `dist/` so the Worker never serves them (`SOURCEMAPS_DELETE=0` keeps them for local `wrangler dev`). Verified locally: 96 maps emitted, no pragma in JS, skip-and-strip clean, bundle policy unchanged at 63 chunks / 1639 KB. Master `6bcf27d`. Three previously-open backlog items closed today via the same pattern — build the engineering half now, plug in account/DSN/spec when it lands. See [[2026-05-26-marietta-na-storybook-visual-regression-session]].

2026-05-26 (late): Bonus closeout — generic HMAC-SHA-256 webhook signature middleware at `api/src/lib/webhook-signature.ts`. Provider-agnostic: `secret` callback (lazy env read → 503 not crash), configurable `header`, `prefix` (GitHub `sha256=`) or custom `extract` (Stripe `t=...,v1=...`), hex/base64 encoding. Verified body stashed on Hono context (one-shot stream — handlers call `getWebhookBody(c)` instead of re-reading). Constant-time compare. 401 mismatch/missing_header/malformed_header, 503 missing_secret. 10 tests pass. Backlog item flipped to `[~]` — lib ready, no inbound provider wired yet. Hono context typing gotcha: `c.set` rejects undeclared keys; cast through `as unknown as { set: (k: string, v: unknown) => void }` rather than leaking a library sentinel onto the global Variables shape. Master at `21aca8a`.

2026-05-26: marietta-na frontend gains Storybook 10 + visual regression across all 17 UI primitives (73 stories, 73 baselines). Stack: `@storybook/react-vite`, `@storybook/test-runner` + `jest-image-snapshot` (1% diff tolerance), `addon-a11y` + axe-playwright for WCAG 2 A/AA. Color-contrast offloaded to existing `gen-contrast-matrix.mjs`. Master-only CI job, parallel with lighthouse + e2e; pre-push hook unchanged so the PR #53 local-first contract holds. Five gotchas captured: (1) Storybook inherits the project's Vite plugin chain — must filter `vite-plugin-pwa` (+ 4 satellites), `@cloudflare/vite-plugin`, `@mdx-js/rollup` via `viteFinal`; (2) `expect` is a Jest global injected at run time, NOT a `@storybook/test-runner` export — register matchers inside the `setup()` lifecycle; (3) `addon-a11y` runs its own check separate from the postVisit hook — disable color-contrast in `preview.ts` `a11y.config.rules`; (4) DataTable filter row inside `<thead>` triggered axe `empty-table-header` on noFilter columns — switched cells from `<Th>` to `<Td>` (real fix, lands a WCAG 1.3.1 improvement); (5) DataTable's `useTranslation` needed a side-effect `import "../src/i18n"` in preview. Planning loop with the Plan agent caught the v9-vs-v10 peer-dep mismatch up front. See [[2026-05-26-marietta-na-storybook-visual-regression-session]].

2026-05-24 (late): PR #53 merged the local-first pipeline pivot on top of the same day's PRs #50-#52. Strategy shift: instead of optimizing PR CI, eliminate it. `.husky/pre-push` runs typecheck + tests + build (~3-5 min) on every contributor's machine before push; PR-time GitHub Actions trigger dropped entirely; master push still runs the four-job backstop (api + frontend + lighthouse + e2e) + Cloudflare Workers Builds deploys from CF's compute. Husky 9 pattern: `prepare: "husky"` in root `package.json`, `.husky/_/` self-gitignored, only `.husky/pre-push` tracked. Net cost cut vs pre-PR-50 baseline: ~60-80% per merged PR. Frontend dist artifact share + Playwright browser cache survive the pivot. Risk: pre-push is the only PR-time gate, so contributors must run `npm install` to pick up the hook. See [[2026-05-24-marietta-na-drive-shortcut-file-kind-session]].

2026-05-24: Cross-project save from marietta-na (Cloudflare Worker + React 19 PWA). Three PRs shipped: #50 (CI Actions cost cut via `dorny/paths-filter@v3` path gating, `frontend/dist` artifact share between frontend/lighthouse/e2e, Playwright browser cache keyed by `@playwright/test` version), #51 (drive shortcut accepts a Drive folder OR a single file — migration 0030 adds `kind` column, `extractDriveItem` replaces `extractDriveFolderId`, picker exposes folders + files views), #52 (top-level `permissions: pull-requests: read` for paths-filter). Two CI gotchas captured: (1) `dorny/paths-filter@v3` 403s without explicit `pull-requests: read` even on PRs from the same repo; (2) module-level `import.meta.env` reads freeze at import time, so any `.env.local`-dependent boolean ships green locally and red in CI — fix is to read inside the component so vitest's `vi.stubEnv` flows through. Vite still inlines `import.meta.env.*` at prod build. Reproducible by `mv .env.local .env.local.bak && npm test`. See [[2026-05-24-marietta-na-drive-shortcut-file-kind-session]].

2026-05-17: Cross-project save from Pippa (personal-finance app, Next.js on Cloudflare Workers). New concept page [[PDF Bank-Statement Parsing on Workers]] captures reusable parsing knowledge. `unpdf` `extractText({mergePages:true})` returns one newline-free blob, so line-based parsers silently return 0; use a global tokenizer with offset-based section sign. PNC layout is Date, Amount, Description with section-set signs. Capital One credit-card layout is wholly different: month-name cycle `Mon DD, YYYY - Mon DD, YYYY`, records `Mon DD Mon DD DESC [- ]$AMT`, signs inverted versus a bank. Cloudflare D1 caps bound params per statement at about 100, so a multi-row INSERT must chunk by columns not rows (otherwise an empty body, and the client sees "Unexpected end of JSON input"). Production account IDs are Plaid-sourced (`acct_plaid_*`) not seeded (`acct_*`); verify backfills against live D1. PII-safe debugging means: pull the real file, run the exact extractor, print only letters-to-a / digits-to-9 shape. Architecture: a `BankStatementParser` interface plus a registry keyed off the schema enum. Address counter unchanged (the save skill does not allocate addresses).

2026-04-28: Cross-project session — set up the bitstub founder-hours weekly tracker as a Claude Code scheduled remote agent (`trig_013b5w322UwGZVhEn3SzRJEA`, Mondays 13:00 UTC, first fire 2026-05-04). Documented the pattern as a wiki concept page at [[Founder-Hours Tracking Routine]] (c-000006). Counter advanced 5 → 6. Page leads with the *pattern* (why solo-OSS projects need an automated weekly tracker — manual logging is itself the failure mode the cap is meant to prevent), then grounds in the bitstub instance. Cross-references [[Persistent Wiki Artifact]] and [[Compounding Knowledge]]. Watch item: bitstub repo private; first fire reveals whether remote-agent env auth pulls a digitalseraph private repo.

2026-04-24 (late night): v1.6.0 public release notes shipped. `docs/releases/v1.6.0.md` (Karpathy-style, 346 lines) establishes the release-notes convention. Three original SVGs at `wiki/meta/dragonscale-{mechanism-overview,6-test-flow,frontier-graph}.svg` carry the visual load; Wikipedia dragon curve referenced by text link only (no binary vendoring). R4 codex verifier ACCEPT WITH FIXES, 3 wording fixes applied. User runs `gh release create v1.6.0 --notes-file docs/releases/v1.6.0.md` when ready. Commits `85515bb` (docs), plus wiki/meta/ auto-commits for SVGs.

2026-04-24 (night): DragonScale end-to-end validation pass. Six-test menu run via Teams orchestration (codex gpt-5.4 for M1 dry-run, M1 commit, M4 autoresearch; chair for ollama pull, M2 allocate, M3 full tiling). All six green. First real fold committed (`wiki/folds/fold-k3-from-2026-04-23-to-2026-04-24-n8.md`, 115 lines, 8 children). First real tiling report at `wiki/meta/tiling-report-2026-04-24.md` (0 errors, 15 review pairs). M2 counter advanced 2 to 3, `c-000002` reserved-unassigned. M4 autoresearch filed 3 new concept pages (`Persistent Wiki Artifact`, `Source-First Synthesis`, `Query-Time Retrieval`) extending `[[How does the LLM Wiki pattern work]]` with Karpathy gist + RAG + MemGPT + Obsidian docs as sources. v1.6.0 validated.

2026-04-24 (evening): v1.6.0 closeout via Teams approach (chair-led, codex gpt-5.4 for sub-agents). 2 explorers (closeout gaps + doc surface). 6 bounded writes (non-overlapping scope): `docs/dragonscale-guide.md` (new, 563 lines), `wiki/meta/2026-04-24-v1.6.0-release-session.md` (new, 346 lines), `wiki/meta/boundary-frontier-2026-04-24.md` (first real M4 run artifact, new), `docs/install-guide.md` (1.5.0 to 1.6.0 + M4 callout + flat-extractive correction), `README.md` (parenthetical + guide link), `wiki/hot.md` (drift fixes). 1 adversarial verifier returned ACCEPT WITH FIXES; all 11 fixes applied in place. Docs commit `eb1562f`. `make test` green (74+ assertions). Still no git tags for v1.5.0 / v1.5.1 / v1.6.0. User requested gpt-5.5; API rejects it on this codex CLI; gpt-5.4 used throughout.

2026-04-24 (late): Phase 4 shipped. Mechanism 4 (boundary-first autoresearch) implemented as `scripts/boundary-score.py` with expanded test coverage. `/autoresearch` without a topic now offers frontier candidates (opt-in, agenda-control labeled). Cross-file status updated. Version bumped to 1.6.0 in `plugin.json` + `marketplace.json`; no git tag created locally (only pre-DragonScale tags `v1.1` - `v1.4.3` exist).

2026-04-24 (afternoon): Phase 3.6 hardening, five surgical fixes (tiling --report path confinement, rollout baseline, AGENTS.md consistency, wiki-ingest .raw contradiction, install-guide version). v1.5.1.

2026-04-24 (morning): Phase 3.5 hardening pass. Cross-phase audit resolved 10 hold-ship items. At that point Mechanism 4 was marked NOT IMPLEMENTED (later reversed in Phase 4 the same day). `bin/setup-dragonscale.sh` + tests + Makefile added, CHANGELOG created, versions synced to 1.5.0.

2026-04-23 (3): Phase 3 complete. Semantic tiling lint shipped as opt-in. `scripts/tiling-check.py` with flock-guarded atomic cache, localhost-locked OLLAMA_URL default, symlink rejection, model-drift invalidation, and banded thresholds (error>=0.90, review>=0.80, conservative seeds). 4 codex review rounds, 10/10 accept.

2026-04-23 (2): Phase 2 complete. Deterministic page addresses MVP via `scripts/allocate-address.sh` (flock-guarded, recovers counter from max observed). New frontmatter `address: c-NNNNNN`. `wiki-ingest` and `wiki-lint` updated with opt-in Address Assignment and Validation sections. 3 codex rounds, 8/8 accept.

2026-04-23 (1): Phase 0-1 complete. DragonScale Memory spec (`wiki/concepts/DragonScale Memory.md` v0.3) plus `skills/wiki-fold/` for Mechanism 1 (log rollups, dry-run verified). Survived multi-round codex review.

## Plugin State

- **Version**: 1.6.0 (Phase 4 shipped; plugin.json + marketplace.json synced; 1.5.1 was the Phase 3.6 hardening point release)
- **Install ID**: `claude-obsidian@claude-obsidian-marketplace`
- **Skills**: 11 (wiki, wiki-ingest, wiki-query, wiki-lint, wiki-fold, save, autoresearch, canvas, defuddle, obsidian-bases, obsidian-markdown)
- **Scripts**: `scripts/allocate-address.sh`, `scripts/tiling-check.py`, `scripts/boundary-score.py` (all opt-in; feature-detected by skills)
- **Setup**: `bin/setup-vault.sh` (base vault), `bin/setup-dragonscale.sh` (opt-in DragonScale), `bin/setup-multi-agent.sh` (multi-agent bootstrap)
- **Tests**: `make test` runs `tests/test_allocate_address.sh`, `tests/test_tiling_check.py`, `tests/test_boundary_score.py`. Zero ollama dependency for core tests.
- **Hooks**: 4 (SessionStart, PostCompact, PostToolUse [stages wiki/, .raw/, .vault-meta/], Stop)

## DragonScale Mechanisms

1. **Fold operator** (Mechanism 1): `skills/wiki-fold/`, dry-run verified AND first real fold committed at `wiki/folds/fold-k3-from-2026-04-23-to-2026-04-24-n8.md`.
2. **Deterministic addresses** (Mechanism 2): shipped and exercised; vault counter at 3. `c-000001` on DragonScale Memory.md. `c-000002` reserved-unassigned from validation pass (gap acceptable per spec).
3. **Semantic tiling lint** (Mechanism 3): shipped and activated. `nomic-embed-text` pulled; first tiling report at `wiki/meta/tiling-report-2026-04-24.md` (0 errors, 15 review-band pairs).
4. **Boundary-first autoresearch** (Mechanism 4): shipped (Phase 4, opt-in). `scripts/boundary-score.py` + `tests/test_boundary_score.py`. `/autoresearch` without a topic surfaces top-5 frontier pages as candidates; user picks, overrides, or declines. Explicitly labeled "agenda control" in both spec and skill.

## Key Lessons from This Release Cycle

1. Cross-phase audits are essential. Individual phase reviews miss drift between phases.
2. Opt-in feature detection (`[ -x script ] && [ -f state ]`) preserves default plugin behavior for adopters and non-adopters alike.
3. PostToolUse hook matcher is `Write|Edit`, so Bash writes don't fire it. Scripts that mutate tracked state must be Bash-only to avoid side-effect commits.
4. Seed-vault self-consistency matters: if the spec says post-rollout pages need addresses, the concept page itself has to have one.
5. Codex adversarial review rounds stop when the punch list is empty, not when the author feels done.

## Style Preferences

- No em dashes (U+2014) or `--` as punctuation. Periods, commas, colons, or parentheses. Hyphens in compound words are fine.
- Short and direct responses. No trailing summaries.
- Parallel tool calls when independent.

## Active Threads

- DragonScale Mechanism 4 shipped in Phase 4 as an opt-in Topic Selection mode in `skills/autoresearch/`. All four DragonScale mechanisms are now shipped and feature-gated.
- v1.6.0 not yet pushed to GitHub (local commits only, no git tag created). User controls push and tag timing.
- CLAUDE.md has one pre-existing uncommitted change ("Release Blog Post" section) that predates this session.

## Repo Locations

- Working: `~/Desktop/claude-obsidian/`
- Public: https://github.com/AgriciDaniel/claude-obsidian
