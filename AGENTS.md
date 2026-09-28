# AGENTS.md

Instructions for AI agents working in this repo. `CLAUDE.md` carries the longer
version of the same material; when they disagree, trust the scripts/config.

## Shape

- npm **workspaces** monorepo: `apps/*`, `packages/*`, `workers/*`. One
  `npm install` at the root wires everything up. **npm only** (the repo tracks
  `package-lock.json`), Node **22+** (`.nvmrc`).
- Two independent Python projects, not part of the npm tree:
  - `backend/geolibre_server` — optional FastAPI sidecar (Whitebox, conversion,
    raster tools). Extras: `conversion`, `vector`, `raster`, `test`.
  - `backend/geolibre_server_api` — projects/identity API. Extras: `test`,
    `postgres`.
  - `python/` — the `geolibre` Jupyter anywidget package (own pytest suite).
- One React app ships three ways: Tauri v2 desktop
  (`apps/geolibre-desktop/src-tauri`), nginx/Docker web build, and bundled into
  the `python/` wheel (`npm run build:embed` → `dist-embed` → `hatch_build.py`).

## Commands

```bash
npm run dev            # web dev server → http://localhost:5173
npm run tauri:dev      # desktop (needed for fs dialogs, local MBTiles/raster)
npm run ci:web         # fast gate, Node only: lint + i18n check + tsc + unit tests
npm run ci             # full gate CI runs: lint + build + all tests + rust
```

- **`npm run typecheck` is not a type check** — it aliases the full
  `tsc -b && vite build` and writes to `dist/`. For a check only, use
  `npm run typecheck:fast` (`tsc -b apps/geolibre-desktop --noEmit`; it must
  first build `@geolibre/embed`, whose `dist/` types the app imports).
- Lint: `npm run lint` (ESLint over `apps packages workers tests`). It carries a
  `--max-warnings` **ratchet**: adding a warning fails CI — fix it or
  `void`/disable with a reason; lower the number when you fix warnings, never
  raise it (except turning on a new rule, by exactly that rule's count).
- Single frontend test: `node --import tsx --test tests/<name>.test.ts`
  (all frontend tests are flat files under `tests/`, not colocated).
- Single backend test: `python -m pytest backend/geolibre_server/tests/test_x.py::test_y`.
  The API project is separate: `python -m pytest backend/geolibre_server_api/tests`
  — **not** covered by `npm run ci`; CI runs it as its own step, and the
  `-m postgres` concurrency tests need a live Postgres
  (`GEOLIBRE_TEST_POSTGRES_URL`).
- Worker gate: `npm run test:worker` typechecks all five workers and also runs
  the `collab-node` unit tests.
- E2E: `npm run test:e2e` (needs `npx playwright install chromium` once; builds
  and serves the app). PRs gate only on the `core` project — the spec list is
  `CORE_SPECS` in `playwright.config.ts`; everything else is `features`
  (nightly / PR label `full-e2e`). Add a spec to `core` only if it would break
  for every user.

## Setup that silently hollows out results

- Install the backend test extra or vector/raster/SQL/ML tests **skip
  themselves** and everything stays green: `pip install -e "backend/geolibre_server[test]"`.
- `backend/geolibre_server/uv.lock` is committed and CI runs
  `uv lock --check --project backend/geolibre_server` — refresh it in the same
  PR as any pyproject change (`uv lock --project backend/geolibre_server`).
- Coverage floors are a **ratchet** (frontend 78/78/63, backend 55). The
  frontend report only counts files a test imports, so the first test for a big
  module can read as a regression — check transitive imports before lowering a
  floor. Read `docs/maintenance.md#coverage-floors`.
- Generated/synced artifacts must be updated in the same PR: run
  `npm run i18n:tools` after adding/renaming processing tools (CI fails on
  drift); keep `skills/geolibre/` and `CITATION.cff` (version = package.json)
  in sync. Details: `docs/maintenance.md#generated-files-and-cross-file-sync`.

## Pre-commit

- Scope it: `pre-commit run --files <paths>`. A local `npm-build` hook runs the
  full production build on every invocation; prefix with `SKIP=npm-build` if you
  already built. `--all-files` churns files you did not touch.
- The `strip-notebook-outputs` hook rewrites **every tracked `.ipynb`** on every
  commit — notebook outputs in your working copy get erased by any commit.
- Formatting is oxfmt (JS/TS/JSON/CSS/YAML/TOML) and ruff (Python/notebooks);
  hooks auto-fix, so commit, re-`git add`, commit again. Oxfmt also reorders
  `package.json` keys. Never format by hand or edit `node_modules`.

## Architecture (non-obvious bits)

- The app is **store-driven**: `@geolibre/core` owns the Zustand store, domain
  types, and the `.geolibre.json` schema. UI changes store state;
  `MapController.syncLayers` (`@geolibre/map`) reconciles MapLibre. **Never
  mutate MapLibre/deck.gl directly from a component.**
- Local vector files become GeoJSON in-browser via DuckDB-WASM Spatial
  (`INSTALL spatial; LOAD spatial;` → `ST_Read`); tile/service/raster layers
  become `GeoLibreLayer` records. Rendering: MapLibre GL JS + deck.gl (also
  CesiumJS for 3D).
- Packages: `@geolibre/core`, `@geolibre/map`, `@geolibre/ui`,
  `@geolibre/processing`, `@geolibre/plugins`, `@geolibre/embed`, plus
  `geolibre-desktop` (shell/Tauri I/O). Built-in plugins live in
  `packages/plugins/src/plugins/` and register in
  `apps/geolibre-desktop/src/hooks/usePlugins.ts`.
- The sidecar is **optional**: Vector tools run client-side with Turf.js and
  only hit `/vector` when the `vector` extra is installed. The browser build
  proxies it at `/sidecar`, confined to `GEOLIBRE_CONVERSION_ROOTS`.

## Conventions

- Never commit to `main`; branch off it and open a PR. Conventional Commits
  (`feat:`, `fix:`, `docs:`, …).
- New user-facing strings go through `t()`; `en.json` is the source of truth.
  **Read `docs/i18n.md` before any i18n work.** The UI mirrors for RTL, so use
  Tailwind logical utilities (`ms-`/`me-`/`ps-`/`pe-`/`text-start`/`border-s`/`start-`),
  never `ml-`/`left-`/`text-right` — ESLint rule `local/no-physical-tailwind`
  enforces it.
- New external tile/style hosts must be added to the Tauri CSP allowlist, or
  desktop loads will silently fail.
- MapLibre control styling: scoped overrides in
  `apps/geolibre-desktop/src/index.css`, never in `node_modules`.
- Hand-mirrored upstream code drifts silently: before bumping `maplibre-gl`,
  any `maplibre-gl-*`, `geolibre-wasm`, or `@tauri-apps/plugin-http` (including
  Dependabot), read `docs/maintenance.md` and run the frontend suite.

## Reference docs

`docs/architecture.md` · `docs/contributing.md` (quality checks) ·
`docs/maintenance.md` (floors, ratchets, generated files) · `docs/i18n.md` ·
`docs/project-format.md` · `docs/plugin-api.md` · `docs/python.md` ·
`docs/mcp.md`
