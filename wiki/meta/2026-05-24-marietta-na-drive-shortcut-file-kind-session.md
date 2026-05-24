---
type: session
title: "2026-05-24 marietta-na drive shortcut file kind + CI cost cut session"
created: 2026-05-24
updated: 2026-05-24
tags:
  - session
  - marietta-na
  - github-actions
  - ci
  - drive-picker
  - cloudflare-workers
status: developing
related:
  - "[[CI Build Gotchas - Cloudflare Workers Vitest Vite]]"
---

# 2026-05-24 marietta-na drive shortcut file kind + CI cost cut session

Three PRs shipped against `digitalseraph/marietta-na` master:

- **#50** — CI Actions minute reduction (path filter, frontend dist artifact share, Playwright browser cache).
- **#51** — Drive shortcut feature: shortcut now points at a Drive folder OR a single Drive file.
- **#52** — `permissions: pull-requests: read` fix for `dorny/paths-filter@v3` 403.

Plus one in-feature hotfix commit (`88aca40`) after PR #51's CI failed because `import.meta.env` reads were frozen at module load.

Branch state ended clean on master; all three feature/CI branches deleted locally.

---

## Feature: Drive shortcut accepts file or folder

### Backend (api/)

**Migration 0030** adds `drive_folder_shortcuts.kind TEXT NOT NULL DEFAULT 'folder'`. Existing rows backfill to `'folder'` correctly (only valid value pre-feature). Column kept as `drive_folder_id` for backwards compat — stores either folder ID or file ID depending on `kind`.

**URL extractor** swap: `extractDriveFolderId` → `extractDriveItem(input, hint='folder')` in `api/src/lib/drive-url.ts`. Returns `{ id, kind: 'folder' | 'file' } | null`.

```ts
// Folder URL → 'folder'
// drive.google.com/drive/folders/{id}
// drive.google.com/drive/u/0/folders/{id}

// File URL → 'file'
// drive.google.com/file/d/{id}
// docs.google.com/{document|spreadsheets|presentation}/d/{id}

// Ambiguous → hint (default 'folder')
// drive.google.com/open?id={id}
// bare ID matching /^[A-Za-z0-9_-]{20,}$/
```

The hint is supplied by the Picker callback's mime read at create time, or by the existing row's `kind` at edit time, so a no-op edit (re-paste of same URL) does not accidentally flip kind.

**Routes** (`api/src/routes/admin/drive-folders.ts` + `api/src/routes/public/drive-folders.ts`):

- POST / PATCH accept optional `kind` field. Persist `kind` from `extractDriveItem` result.
- Public `GET /drive-folders` returns `kind` + a kind-aware `driveUrl` (`/file/d/{id}/view` for files, `/drive/folders/{id}` for folders).
- Public `GET /drive-folders/:id/contents` returns `400 not_listable` when row is file kind so the frontend never spin-loads a non-existent file listing.
- Admin `GET /:id/preview` short-circuits file kind (no Drive API call needed).

### Frontend (frontend/)

**Picker** (`frontend/src/lib/google-picker.ts`) — Two `DocsView`s now:

```ts
const foldersView = new pickerNs.DocsView(pickerNs.ViewId.FOLDERS)
  .setSelectFolderEnabled(true)
  .setIncludeFolders(true)
  .setMimeTypes("application/vnd.google-apps.folder");
const filesView = new pickerNs.DocsView(pickerNs.ViewId.DOCS)
  .setSelectFolderEnabled(true)
  .setIncludeFolders(true);
```

User switches tabs inside the picker chrome to pick a folder or any file. Callback derives `kind` from `doc.mimeType === "application/vnd.google-apps.folder"`. Exports `PickedItem` with `kind` field; `PickedFolder` retained as a back-compat alias.

**Admin page** (`AdminDriveFoldersPage.tsx`) — Renders a Folder/File badge per row. Form forwards `kind` to the API as a hint. `getPickerConfig()` reads `import.meta.env` at render time so vitest's `vi.stubEnv` works (see CI failure below).

**Documents page** (`DocumentsPage.tsx`) — Folder shortcuts keep the `Browse → /documents/library/:id` inline listing flow. File shortcuts render as a direct `<a target="_blank">` to Drive — no library page (nothing to list).

### Test coverage

- `extractDriveItem`: folder URL / file URL / docs URL / ambiguous `open?id=` / bare ID with hint
- POST stores file kind end-to-end
- Public list returns kind + correct driveUrl shape
- `/:id/contents` rejects file kind
- Admin page test forwards `kind: 'file'` and shows File badge after submission
- 332 api + 234 frontend tests pass

---

## CI cost reduction (PR #50)

User reported 1,832 / 3,000 GitHub Actions minutes used for the month. Three optimizations targeted minute usage without dropping any quality gate:

**1. Path filter** via `dorny/paths-filter@v3` in a leading `changes` job:

```yaml
filters: |
  api:
    - 'api/**'
    - '.github/workflows/**'
  frontend:
    - 'frontend/**'
    - 'docs/**'
    - '.github/workflows/**'
```

Downstream jobs gate on outputs. Master push always runs everything. Doc-only PRs skip frontend / lighthouse / e2e. API-only PRs skip frontend-side jobs.

**2. Frontend dist artifact share** — Frontend job uploads `frontend/dist` (1-day retention) after all builds (SPA + docs/site + docs/presentation copied in). Lighthouse + e2e download instead of running `npm run build` themselves. Saves ~3 min per job × 2 jobs per CI run.

**3. Playwright browser cache** keyed by `@playwright/test` package version:

```yaml
- name: Resolve playwright version
  id: pw
  run: |
    v=$(node -p "require('@playwright/test/package.json').version")
    echo "version=$v" >> "$GITHUB_OUTPUT"
- uses: actions/cache@v4
  with:
    path: ~/.cache/ms-playwright
    key: pw-${{ runner.os }}-${{ steps.pw.outputs.version }}
```

Cache-hit branch runs `playwright install-deps chromium` (OS libs only); cache-miss branch runs full `playwright install --with-deps chromium`. Saves 30-60s per warm e2e run.

**Expected savings per CI run:**
- Doc-only PR: ~10 min → ~30s
- API-only PR: ~10 min → ~2 min
- Frontend PR: ~10 min → ~7 min
- Master push: unchanged total wall clock; PR savings only

---

## Gotcha 1: dorny/paths-filter needs explicit pull-requests:read

First run of new CI workflow on PR #51 failed at the `changes` job:

```
Resource not accessible by integration
```

`dorny/paths-filter@v3` queries the GitHub REST API for the PR's changed file list. The default `GITHUB_TOKEN` scope in this repo did not include `pull-requests: read`. Fix in PR #52 — add explicit top-level permissions:

```yaml
permissions:
  contents: read
  pull-requests: read
```

Read-only, scoped to exactly what the action needs. Reproducibly affects repos where the default token scope is restricted at the org/repo settings level.

---

## Gotcha 2: `import.meta.env` reads freeze at module load (CI vs local)

PR #51's CI failed after PR #52 unblocked the path filter: six picker tests in `AdminDriveFoldersPage.test.tsx` could not find the Pick from Drive button:

```
Unable to find an accessible element with the role "button"
and name `/pick from drive/i`
```

Tests passed locally. Root cause: local `frontend/.env.local` (gitignored) has `VITE_GOOGLE_CLIENT_ID` + `VITE_GOOGLE_API_KEY` + `VITE_GOOGLE_APP_ID` set, so module-level `PICKER_CONFIGURED` evaluated to `true` at import time. CI has no `.env.local`, so the module-level constant froze at `false` before vitest's `vi.stubEnv` calls could take effect — and the button is conditionally rendered on `PICKER_CONFIGURED`.

ES module imports are hoisted; `vi.stubEnv` is not. So in the test:

```ts
vi.stubEnv("VITE_GOOGLE_CLIENT_ID", "test-client-id"); // runs second
import { AdminDriveFoldersPage } from "./AdminDriveFoldersPage"; // hoisted, runs first
```

Local was masking the bug. Fix in commit `88aca40`: move env reads behind a `getPickerConfig()` helper called from inside the React component, so each render re-reads `import.meta.env`:

```ts
function getPickerConfig() {
  return {
    clientId: import.meta.env.VITE_GOOGLE_CLIENT_ID ?? "",
    apiKey: import.meta.env.VITE_GOOGLE_API_KEY ?? "",
    appId: import.meta.env.VITE_GOOGLE_APP_ID ?? "",
  };
}

function UI() {
  const pickerConfig = getPickerConfig();
  const pickerConfigured = Boolean(pickerConfig.clientId && pickerConfig.apiKey);
  // ...
}
```

Vite inlines `import.meta.env.*` at production build, so the prod bundle still resolves to literal string constants — no runtime cost beyond a stable function call.

**Local reproduction:** rename `.env.local` aside before running `npm test` to simulate the CI environment:

```bash
mv .env.local .env.local.bak
npm test
mv .env.local.bak .env.local
```

---

## Workflow notes

- Mid-stream context switch (user shifted from drive feature to CI cost) handled by committing drive WIP on its branch with no push (zero Actions cost), then branching `ci/cut-actions-costs` off master cleanly. Two separate PRs let CI changes merge first so the drive PR ran on the optimized workflow.
- `wrangler d1 migrations apply` scans `drizzle/*.sql` by filename; the `drizzle/meta/_journal.json` (drizzle-kit's diff-generation state) is not consumed by `wrangler` and is allowed to lag.
- Frontend i18n parity test only requires other locales to be a subset of `en`. Adding new EN keys is non-breaking; ES/FR/PT translations fall back through i18next.
- Bundle policy: 63 chunks / 1620 KB, well under cap.

---

## Cross-project references

This is the first wiki page filed for the marietta-na project. Likely future links:

- `[[Marietta NA]]` — project entity page (not yet created)
- `[[MASCNA Policy]]` — fellowship policy doc the schema encodes
- `[[Drive Folder Shortcuts]]` — feature concept (not yet promoted from session)
