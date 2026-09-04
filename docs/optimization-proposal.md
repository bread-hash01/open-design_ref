# Open Design — Whole-Repository Optimization Proposal

**Status:** Proposal v1 · 2026-09-04
**Parent:** [`spec.md`](spec.md) · **Siblings:** [`architecture.md`](architecture.md) · [`design-systems-token-audit.md`](design-systems-token-audit.md) · [`code-review-guidelines.md`](code-review-guidelines.md) · [`roadmap.md`](roadmap.md)
**Scope:** Every component in the workspace — 7 apps, 12 packages, 3 tools, `e2e/`, root `scripts/`, CI, and the four content registries (`skills/`, `craft/`, `design-systems/`, `design-templates/`), plus `mocks/` and `docs/`.

---

## 0. How to read this

Twelve parallel read-only audits measured each component against its own `AGENTS.md`. Every claim below carries a file path, a line number, or a count that was observed on branch `docs/design-resource-reference`. Nothing here has been changed in the repo; this document is the plan, not the work.

Findings are labelled by component (`WEB-F1`, `DMN-F3`, …) so a PR can cite one. Impact/effort use the audits' own H/M/L and S/M/L. §4 sequences everything into five waves; if you read only one section, read that one.

Two corrections to assumptions that were in circulation before this audit:

- **`apps/landing-page` is Astro 6.3.5, not Next.js.** It is fully static, has zero `client:*` directives, and renders React only at build time via `renderToStaticMarkup`. Any plan written against `next/image`, `use client`, or ISR does not apply to it.
- **The system prompt contains no skill catalogue.** Only the *active* skill body, its craft bundle, and the active design system are injected. Catalogue size costs HTTP and disk, not tokens; skill and design-system *bodies* cost tokens. This changes where token optimization pays off.

---

## 1. Executive summary

The repository is structurally sound and operationally expensive. Guards, boundaries, and stamps hold almost everywhere they are checked; the cost sits in three places nothing checks: **per-turn token spend**, **per-request filesystem work**, and **per-PR build time**.

The ten highest-value items, in rough value order:

| # | Item | Measured today | Component |
|---|---|---|---|
| 1 | Design-system injection is uncapped | mean **7,031 tokens/turn**, p90 11,712, max 19,212 (`stripe`) | `design-systems/` |
| 2 | The 15 unmigrated design systems inject raw HTML fixtures | **10,195 vs 6,680 tokens** (+53%) | `design-systems/` |
| 3 | Catalog scans are uncached, per request | **1,066 fs ops / 1.2 MB / ~74 ms** per call, from 34 call sites | `apps/daemon` |
| 4 | Every SSE token delta does a read-modify-write of SQLite | O(n²) in events per message, synchronous in `stdout.on('data')` | `apps/daemon` |
| 5 | mac ships beta identity for ad-hoc namespaces | `beta-local-flow` → `Open Design Beta.app` | `tools/pack` |
| 6 | CI rebuilds every package in every job | ~**90 package builds per PR**, no `dist` cache | CI |
| 7 | `pnpm typecheck` runs 40 cold compilers | 37 `tsc` + `astro check` + 3 side-effect builds, zero project references | workspace |
| 8 | Craft rules can cost 90× the skill that requires them | 698 B body + **63,617 B** of rules | `craft/` |
| 9 | Four unused deps in `apps/web` | ~**43 MB / 3,970 files** per install | `apps/web` |
| 10 | 349 of 920 contract exports have no consumer | 38% of `packages/contracts` | `packages/contracts` |

Two of these are correctness issues rather than efficiency ones and should be treated as bugs: **#5** (channel identity) and the ungated Electron IPC handlers (`DSK-F1`).

---

## 2. Cross-cutting themes

### 2.1 Token spend is unbudgeted

Nothing in the prompt path measures itself. Per turn, before the user's own words:

- design system: 28,126 B mean → **7,031 tokens** (`apps/daemon/src/prompts/system.ts:619-676`)
- active skill: 5,7 KB mean → ~1.4k tokens; `taste-skill` alone is 87,915 B → **~26.5k tokens** with its craft bundle
- craft rules: up to 63,617 B for a 698 B skill (`design-templates/mobile-onboarding`)

A design-template turn using `mobile-onboarding` + a p90 design system spends roughly **28k tokens of context** before anything task-specific. The fix is not deletion but tiering: a `## Core` slice injected by default and a `references/` tail pulled on demand, plus a byte budget in `composeSystemPrompt` that spills the DESIGN.md tail to the already-existing, under-used pull channel.

### 2.2 The same work is repeated per request, per job, per install

| Layer | Repetition | Evidence |
|---|---|---|
| HTTP | full catalog scan to resolve one asset path | `routes/static-resource.ts:569-597` |
| Prompt | `loadAllSkills()` + `listAllDesignSystems()` called twice per run | `server.ts:10505/10522`, `10647/10651` |
| SQLite | statement re-compiled per call (74 `prepare` in function bodies) | `db.ts` |
| Pack | workspace build implemented three times; Linux builds two packages twice | `mac/workspace.ts:10-44`, `win/app.ts:117-145`, `linux.ts:389-431` |
| CI | 15 packages rebuilt in 6 jobs | `scripts/postinstall.mjs:9-25` |
| Typecheck | shared packages re-checked N times, tests configs re-include `src/**` | `apps/daemon/tsconfig.tests.json:8-10` |

### 2.3 God files concentrate the risk

| File | LOC | Note |
|---|---|---|
| `apps/daemon/src/server.ts` | 15,011 | `@ts-nocheck`; 188 inline routes; `startChatRun` is 2,554 LOC |
| `apps/web/src/components/FileViewer.tsx` | 10,090 | `HtmlViewer` ≈4,900 LOC, 44 `useState`, 51 `useEffect` |
| `apps/daemon/src/cli.ts` | 7,959 | `@ts-nocheck`; 170 top-level functions; top-level-await dispatch |
| `apps/web/src/components/SettingsDialog.tsx` | 7,359 | 9 sections, statically imported from 5 places |
| `apps/web/src/components/ProjectView.tsx` | 6,322 | 30 props drilled from `App.tsx` |
| `apps/daemon/src/media.ts` | 3,848 | 30-branch provider `else if` |
| `apps/desktop/src/main/runtime.ts` | 1,976 | URL policy + HMAC + PDF + pet window + splash |
| `tools/pack/src/linux.ts` | 1,661 | one file against mac's 18 and win's 20 |

`@ts-nocheck` covers **27,380 LOC — 21% of daemon `src/`**. That is TypeScript in extension only, and it is where the untyped `(req, res)` handlers and the 345 `any` live.

### 2.4 Duplication is systematic, not incidental

`escapeRegExp` ×7 · relative-time formatters ×6 · run-status predicates ×4/×3 · SSE parsers ×4 (with divergent `\r` handling) · JSONL splitters ×4 + a fifth variant (one missing an `isRecord` array guard) · `gotoEntryHome` ×24 in `e2e/ui` despite being exported from `lib/` · `pathExists` ×5 and `runPnpm` ×4 in `tools/` · channel-identity ladders ×2 (divergent) · button CSS ×3 (`components/styles.css`, `web/styles/primitives.css`, `button.module.css`, 44 identical selectors) · deck runtime pasted ≥31× with 8 byte-identical `deck-stage.js` copies.

Divergence has already produced at least three real defects: the qoder JSONL guard, the mac channel matcher, and the two locale-code systems in the landing page.

### 2.5 Dead weight is measurable and safe to remove

349 unused contract exports · 98 unused web exports · ~545 orphan global CSS classes · 4 unused web deps (~43 MB) · Tailwind running through PostCSS with zero directives · 176 of 179 mock traces unreferenced · 17.2 MB of images = 88% of `docs/` · duplicate `/artifacts` and `/frames` static mounts · `components/src/primitives.tsx` byte-identical to `index.ts`.

### 2.6 Validation covers shape, not truth

CI runs **5 of 11** package test suites. There is no content lint (no check for `od.mode` enum, craft-slug existence, side-file resolution, or size budgets), no link checker (127 broken relative links, 111 foreign `/Users/mac/...` paths), no CLI↔API parity test despite parity being a mandated shipping gate, no migration test, no ESLint anywhere (30 `eslint-disable` comments are inert), and no purity test enforcing the contracts boundary that currently holds by luck.

---

## 3. Component-by-component

### 3.1 `apps/web` — runtime (310 files, 142,427 LOC excl. CSS/i18n)

Next.js 16 App Router shell hosting a client-only React 18 SPA. 40 files >800 LOC, 18 >2,000. Tests: 312 files / 87,521 LOC / 2,965 cases.

- **WEB-F1 · Split `HtmlViewer`** (H/L). `FileViewer.tsx:4406-9303`; the JSX return starts 3,510 lines in. Carve `useCommentBoard()`, `useManualEditInspector()`, `useDeployShare()`, `InspectPanel`, and the pure inspect-override functions (3160–3520) into `src/components/file-viewer/` + `src/runtime/inspect-overrides.ts`.
- **WEB-F2 · A store for app catalogs** (H/L). `ProjectView.tsx:745-5741` takes 30 props from `App.tsx:1898-1932`; only 2 contexts exist repo-wide. Move `agents`/`skills`/`designSystems`/`projects`/`templates` behind a `useSyncExternalStore` store in `src/state/`, one catalog at a time.
- **WEB-F3 · Split + lazy-load `SettingsDialog`** (M/M) — 7,359 LOC statically imported from five call sites, pulling `ConnectorsBrowser` (1,563) and `MemorySection` (2,441) into the first chunk.
- **WEB-F4 · De-duplicate helpers** (M/S) — `src/utils/strings.ts`, `src/utils/time.ts`, `src/runtime/run-status.ts`. The run-status predicates matter most: divergent copies decide whether a Stop button renders.
- **WEB-F5 · One SSE reader** (M/S) — `registry.ts:118-160` and `state/projects.ts:1328-1360` hand-roll parsers that ignore `\r`; route both through `providers/sse.ts`.
- **WEB-F6 · `apiRequest<T>()`** (M/M) — 139 `fetch(` sites outside `providers/`; 80 copies of `if (!resp.ok)` inside `registry.ts`.
- **WEB-F7 · Replace polling with events** (M/S) — `App.tsx:1481` polls `listProjectRuns()` every 2 s and restarts the timer on every `projects` change; `ProjectView.tsx:3781` polls Vela login for 5 min.
- **WEB-F8 · Delete dead exports and deps** (M/S) — 98 orphan exports; `openai`, `@formkit/auto-animate`, `lucide-react` unused.
- **WEB-F10 · Add `eslint-plugin-react-hooks`** (M/S) — package-scoped; 78 effects in one file with no backstop.
- **WEB-F12 · Typed bridge scripts** (M/L) — `runtime/srcdoc.ts` holds ~570 lines of stringly JS invisible to `tsc`.
- **WEB-F14 · `React.lazy` six secondary screens** (M/S).

### 3.2 `apps/web` — CSS, i18n, build (54,245 CSS LOC; 19 locales)

- **WEB-C1 · Lazy locale loading** (H/M) — `src/i18n/index.tsx:12-30` value-imports all 19 locales (4.03 MB) and `content.ts:7-127` imports 16 content files (2.4 MB); an English user downloads every string ×19. `t()` already falls back through `DICTS[locale] ?? en`, so the shape supports it.
- **WEB-C3 · Subset RemixIcon** (M/S) — 157 KB CSS + 189 KB woff2 imported first in the cascade for **33 icons**.
- **WEB-C4 · Self-host Cairo** (M/S) — `index.css:1` blocks render on `fonts.googleapis.com` for one `[dir="rtl"]` rule; packaged builds run offline.
- **WEB-C5 · Remove Tailwind** (L/S) — the PostCSS plugin runs on every file with zero `@tailwind`/`@apply`/`@theme` directives anywhere.
- **WEB-C7 · Fold `styles/home/index.css` into the cascade** (H/L) — 11,001 LOC imported from `app/layout.tsx:6`, outside the `index.css` entrypoint AGENTS.md mandates, and invisible to the 15 `tests/styles/*` specs.
- **WEB-C10/C13 · Motion and z-index tokens** (M/S) — the documented bezier is pasted 372 times, bare `ease` appears 434 times, and 51 distinct z-index literals exist against one token. (`ease-in` is at 0 — keep it there with a lint.)
- **WEB-C15 · Legacy primitives** (M/L) — `ghost` 105 uses / 27 files, `primary` 57 / 23, `icon-btn` 14 / 9. Add a ratchet counter, migrate by file.
- Doc corrections: AGENTS.md says 18 locales (there are 19; `it` is missing from the list) and places `.accordion-collapsible` in `index.css` (it lives in `styles/viewer/composio.css:721`).

### 3.3 `apps/daemon` — server, routes, runtimes, streams (315 files, 128,121 LOC)

- **DMN-F1 · Batch message-event persistence** (H/M) — `server.ts:11562 → 2706 → db.ts:1323` does SELECT + parse + stringify-compare + spread + stringify + UPDATE per token delta. Buffer per run, flush in one transaction on a 250–500 ms timer and on close.
- **DMN-F2 · SSE backpressure** (M/S) — `runs.ts:136` re-serialises per client and `server.ts:4220` ignores `res.write()`'s return; no `drain` handling exists anywhere in `src`.
- **DMN-F3 · Extract `startChatRun`** (H/L) — 2,554 LOC with 15 `def.id === '…'` special cases and an if/else streamFormat dispatch at `12935-13112`.
- **DMN-F4 · Retire `@ts-nocheck`** (H/M) — start with `runs.ts` (349 LOC), give `HttpDeps` real signatures (`server-context.ts:7-13` types every member as `(...args:any[])=>any`).
- **DMN-F5 · One JSONL reader** (M/S) — four copies plus `acp.ts`'s `split('\n')`; `qoder-stream.ts:14` is missing the array guard the others have.
- **DMN-F7 · `defineAcpAgent()`** (M/S) — 9 defs differ by three lines.
- **DMN-F9 · Cap stderr frames** (L/S) — uncapped stderr can evict the 2,000-event replay window.
- **DMN-F10/F11 · Test the registrars and the mocks** (M/M) — `project-routes.ts` (44 routes) has zero direct tests; only 3 of 179 mock traces are asserted.

### 3.4 `apps/daemon` — CLI, loaders, storage (~30k LOC of the same tree)

- **CLI-F1/F2/F3 · Catalog index service** (H/M) — one mtime-invalidated index behind `list({includeBody:false})` and `resolve(id)`. Removes ~1.2 MB and ~1,000 fs ops from asset requests, and stops `/api/design-systems` reading 1.6 MB to return metadata.
- **CLI-F5 · `od media generate` parity** (H/S) — no `--prompt-file`, no `--json`, against an AGENTS.md rule that names both. Only 3 of 27 handlers implement `--prompt-file` at all.
- **CLI-F6 · Surface hidden domains** (M/M) — `deploy` (6 routes, 2,002 LOC), `xai`, `codex-pets`, `integrations`, `applied-plugins`, `orbit`, `prompt-templates` have no subcommand; `connectors` (12 routes) and `live-artifacts` (14) are dispatched outside `SUBCOMMAND_MAP` and absent from help.
- **CLI-F7 · Delete duplicate static mounts** (L/S) — `/artifacts` at `server.ts:6008` and `:9140`; `/frames` at `:6035` and `:9480`.
- **CLI-F8 · Versioned migrations** (M/M) — `db.ts:54` re-derives schema state from 10 `PRAGMA table_info` calls and ~25 conditional `ALTER TABLE`s at every boot, with no `user_version` ledger, no ordering guarantee, and no partial-apply detection.
- **CLI-F9/F10 · Statement cache and transactions** (M/M) — 74 `prepare()` in function bodies; exactly 2 transactions for 13 tables.
- **CLI-F12 · Lazy CLI imports** (L/S) — every `od` invocation parses 16 files / 585 KB, including the 128 KB connectors CLI.
- **CLI-F14 · Parity spec in `e2e/tests/`** (M/M) — enumerate `SUBCOMMAND_MAP` against the 246-route table; the mandated invariant currently has no test.

### 3.5 `apps/desktop` / `apps/packaged` / `apps/telemetry-worker`

- **DSK-F1 · Gate every privileged IPC handler** (M/S) — `requireMainWindowSender` covers 3 channels; `shell:open-external`, both folder pickers, `shell:open-path`, `od:print-pdf`, `diagnostics:export-to-file` are ungated while `od:` child windows inherit the preload.
- **DSK-F2 · CSP + permission handler** (M/S) — neither exists; `webviewTag: true` at `runtime.ts:1517`.
- **DSK-F3 · Splash before the auth handshake** (H/S in dev) — worst case **8.1 s** with no window; packaged already does it right.
- **DSK-F8 · Parallel sidecar boot** (H/M) — daemon and web start sequentially though web needs only `OD_PORT`; `allocatePort` already exists.
- **DSK-F10 · Timeout the updater fetches** (M/S) — a hung metadata fetch leaves `tickRunning = true` forever, silently ending all future update checks.
- **TEL-F12 · Harden the relay** (M/S) — no upstream timeout; the client rate-limit key is a client-supplied `userId`; no server-side PII redaction; the worker is in no CI workflow.
- **DSK-F7 · Stop duplicating across the app boundary** (M/S) — promote the uncaught-exception filter to `packages/platform` and `createPackagedStamp` to `packages/sidecar` rather than importing across apps.

### 3.6 `apps/landing-page` (Astro 6.3.5, 231 files, 62,391 LOC)

- **LND-F1 · CSS delivery** (H/M) — `inlineStylesheets: 'always'` inlines ~160 KB into each of ~9k generated pages.
- **LND-F2 · One i18n system** (H/L) — a modern short-code system and a legacy BCP-47 one coexist; the catch-all builds `/zh-CN/*` pages that `_redirects` then 301s away.
- **LND-F3/F4 · Token fidelity** (M/M) — `globals.css:26-47` is the real Atelier Zero palette while the design system ships a stub; the same hexes are re-hardcoded in the Shiki theme and `theme-color`; `#fff` ×11 and `#000` ×10 against a DESIGN.md that forbids both.
- **LND-F5 · Remove the hard-coded PostHog key** (M/S) — `_lib/posthog-analytics.ts:19` ships `phc_…` as a fallback, so the tracker loads even without the env var.
- **LND-F6 · Stop the per-view GitHub API call** (M/S) — the build already resolves the star count.
- **LND-F9 · Run the one test that exists** (H/M) — no workflow runs it; `_lib/catalog.ts` (1,160 LOC) and the 100-line redirect map are untested.

### 3.7 `packages/*` (12 packages)

- **PKG-F1 · Prune contracts** (H/M) — 349 of 920 exports unreferenced; only 143 (16%) are genuine two-sided contracts.
- **PKG-F5 · Move `prompts/` to the daemon** (M/M) — 2,187 LOC of prompt prose, 15 of 16 exports daemon-only, currently shipped in the web bundle.
- **PKG-F8 · De-triplicate button CSS** (M/M) — 44 identical selectors across three files kept apart only by cascade order.
- **PKG-F13 · Fix `metatool` packaging** (L/S) — three tools reach into its private `src/` by relative path; it is in no manifest and no build list.
- **PKG-F15 · Run all package suites in CI** (H/S) — 6 of 11 never run, including plugin-runtime's 43 cases.
- **PKG-F4 · Enforce contracts purity with a test** (M/S) — the boundary holds today by convention only.

### 3.8 `tools/dev`, `tools/pack`, `tools/serve`

Boundary compliance is clean: zero hand-built stamps, zero process-scan regexes, zero `any`/`@ts-ignore` in all three tools.

- **TLS-F6 · One channel-identity resolver** (H/S) — **correctness bug**: `mac/identity.ts:18` uses a loose `(^|[-_.])beta($|[-_.])` regex where `win/identity.ts:18` requires exactly `release-beta-win`, so a test namespace like `beta-local-flow` ships as `Open Design Beta.app` with the beta `appId` — precisely what `tools/pack/AGENTS.md` forbids.
- **TLS-F7 · Give Linux a channel identity** (M/S) — `linux.ts:37-38` hard-codes `Open Design` with no channel branch.
- **TLS-F1/F5 · One workspace build** (M/S, and it fixes a stale-cache bug) — three copies; the cache key hashes a `BUILD_COMMANDS` array that is never executed, so editing a real build step can produce a cache hit. Linux additionally builds `host` and `download` twice.
- **TLS-F2/F3 · Cache parity** (H/M) — Linux has zero cache nodes against Windows' eight; mac wipes builder output and re-packs every tarball on every run.
- **TLS-F4 · Level the 16 serial builds** (H/M) — ~11 are independent.
- **TLS-F9 · Error handler and honest `--json`** (M/S) — `tools-pack` prints JSON unconditionally and has no `unhandledRejection` handler, unlike its two siblings.
- **TLS-F8 · A dispatch test** (M/M) — covers all 26 subcommand strings in one test.

### 3.9 `e2e/`, `scripts/`, CI

- **CI-F1 · Build once** (H/M) — ~90 package builds per PR; no `dist` caching.
- **CI-F2 · Use the parallel daemon config** (H/S) — it exists and is used only by the self-hosted runner.
- **CI-F4 · Parallelize Playwright** (H/L) — 240 tests at `workers: 1`, compensated by re-installing lanes.
- **CI-F6 · Move the 24 copied helpers into `lib/playwright`** (M/S).
- **CI-F9 · Re-enable Cachix** (H/S) — `nix flake check` builds cold on every scoped PR with the cache commented out.
- **CI-F10 · TS project references** (M/M) — 35–40 `tsc` runs, 14 packages checking `src/**` twice.
- **CI-F11 · Pin actions to SHAs** (M/S) — 230 tag refs, 0 pins, including `lowlighter/metrics@latest`.

### 3.10 `skills/`, `craft/`, `design-templates/`

- **CNT-F3 · Tier the craft rules** (H/M) — 63.6 KB of rules for a 698 B skill; `laws-of-ux` (17.3 KB) and `form-validation` (17.2 KB) are the heaviest and least used.
- **CNT-F1/F2/F13 · Split oversized skill bodies** (H/M) — `taste-skill` 87.9 KB; the three imagegen/image-to-code skills 36–41 KB each with ~60% overlap; the GSAP family embeds 3–4 KB of code fences per body.
- **CNT-F7 · Shrink `open-design-landing`** (H/S) — 22.7 MB of PNGs, copied into the project cwd on **every turn** by `stageActiveSkill()`, and shipped in the packaged app.
- **CNT-F6 · Fix 88 unresolvable side-file references** (H/M) — 15 templates point at a sibling's `assets/`, 24 still cite pre-split `skills/html-ppt/...` paths.
- **CNT-F4 · Five phantom craft slugs** (M/S) — `typographic-rhythm`, `pixel-discipline`, `motion-discipline` are required but do not exist and are dropped silently.
- **CNT-F8 · A shared deck runtime** (M/L) — 31 hand-rolled navigation implementations, 8 byte-identical `deck-stage.js` copies.
- **CNT-F12 · `scripts/lint-content.ts`** (H/M) — the single highest-leverage addition: required fields, `od.mode` enum, craft-slug existence, side-file resolution, `example.html`, size budgets, run from `pnpm guard` with a shrinking allowlist.

### 3.11 `design-systems/`, `mocks/`, `docs/`

- **DSY-F1/F2 · Finish the backfill, then budget** (H/M) — the 15-system block costs +53% tokens; then cap `composeSystemPrompt` and spill to the pull channel.
- **DSY-F4/F5 · Categories and index** (M/S) — 20 category strings with near-duplicates, 15 systems uncategorized, and a hand-written README claiming 133 systems against 150 shipped.
- **MCK-F8 · `.gitignore` `mocks/recordings/`** (M/S) — one `git add -A` currently commits 4 MB of fetched fixtures.
- **MCK-F9 · Anonymization provenance** (M/M) — `anonymization_version` is null on all 179 entries, and 170 `user_input_preview` fields carry verbatim user prompts including four real business URLs and a named customer project.
- **MCK-F6/F7 · Make the corpus assert something** (M/M) — 3 of 179 traces are pinned; 11 shipped mock CLIs have no trace at all, and two agents have traces but no bin.
- **DOC-F10 · Move `reference/shots/` to R2** (H/M) — 10.3 MB of 105 JPEGs; reuse the mocks manifest + fetch pattern. Cuts `docs/` from 19.1 MB to ~8.8 MB.
- **DOC-F11/F12 · Fix 111 foreign paths and 127 broken links** (M/S–M), then add a link checker to `pnpm guard`.
- **DOC-F13/F14 · Stop committing generated reports; generate the index** (M/S) — three daily cron PRs rewrite 24.6 KB of state whose newest entry is 107 days stale; 23 of 40 top-level docs are missing from the AGENTS.md index, including the design-system contract itself.

### 3.12 Dependencies and build graph

- **DEP-F1/F2 · Delete 5 unused deps** (H/S) — ~43 MB, ~3,970 files.
- **DEP-F3/F4/F5 · One toolchain via pnpm `catalog:`** (H/M) — TypeScript is split 11/11 across a graph where a 5.9.3 consumer reads 6.0.3 declarations; `@types/node` spans 18/20/24 on a Node-24 runtime.
- **DEP-F6/F7 · Project references** (H/L) — 39 tsconfigs, zero `composite`, zero `references`, no shared base, targets split ES2022/ES2024.
- **DEP-F9 · Derive the postinstall order from the graph** (H/M) — a hand-maintained 15-entry list rebuilt serially on every install.
- **DEP-F10 · Declare `metatool`** (M/S). **DEP-F11 · Delete the duplicated `pnpm` block** (M/S). **DEP-F12 · Guard `onlyBuiltDependencies`** (M/S).

---

## 4. The program

Five waves. Each wave is independently shippable; later waves depend on earlier ones only where stated.

### Wave 0 — Correctness and hygiene (days, not weeks)

Ship these first; they are small, they are bugs or leaks, and they do not block anything.

| Item | Component | Effort |
|---|---|---|
| Strict channel-identity resolver, mac + win + linux (`TLS-F6`, `TLS-F7`) | tools/pack | S |
| Gate the seven ungated IPC handlers; strip preload on child windows (`DSK-F1`) | desktop | S |
| `AbortSignal.timeout` on updater and telemetry fetches (`DSK-F10`, `TEL-F12a`) | desktop, worker | S |
| Remove the hard-coded PostHog key (`LND-F5`) | landing-page | S |
| `.gitignore mocks/recordings/` (`MCK-F8`) | mocks | S |
| Purge 111 `/Users/mac/...` paths + guard regex (`DOC-F11`) | docs | S |
| Delete duplicate `/artifacts` + `/frames` mounts (`CLI-F7`) | daemon | S |
| `od media generate --json --prompt-file` (`CLI-F5`) | daemon | S |
| Delete 5 unused deps + the duplicated `pnpm` block (`DEP-F1/F2/F11`) | workspace | S |

Note: the dependency deletions change `pnpm-lock.yaml` and therefore require the Nix hash refresh.

### Wave 1 — Cost reduction on the hot paths (highest measured value)

| Item | Component | Impact | Effort |
|---|---|---|---|
| Catalog index service (`CLI-F1/F2/F3`) | daemon | H | M |
| Batch SSE→SQLite persistence (`DMN-F1`) | daemon | H | M |
| Finish the 15-system 2.0 backfill (`DSY-F1`) | design-systems | H | M |
| Craft tiering: `## Core` + `references` (`CNT-F3`) | craft | H | M |
| Split `taste-skill` and the imagegen family (`CNT-F1/F2`) | skills | H | M |
| Compress `open-design-landing` assets + staging allowlist (`CNT-F7`) | design-templates | H | S |
| Statement cache + transactions (`CLI-F9/F10`) | daemon | M | M |
| Lazy locale loading (`WEB-C1`) | web | H | M |
| Subset RemixIcon, self-host Cairo, drop Tailwind (`WEB-C3/C4/C5`) | web | M | S |

Expected: median turn drops by roughly a third of its pre-prompt tokens; asset requests stop doing ~1,000 fs ops; first paint loses ~340 KB plus 18 locales' worth of strings.

### Wave 2 — Build and CI throughput

Order matters: `DEP-A` (catalog) → `DEP-B` (project references) → `CI-F1` (build-once) → `CI-F10`.

| Item | Component | Impact | Effort |
|---|---|---|---|
| pnpm `catalog:` for the toolchain (`DEP-F3/F4/F5`) | workspace | H | M |
| TS project references + `tsconfig.base.json` (`DEP-F6/F7`, `CI-F10`) | workspace | H | L |
| Derive postinstall order from the graph (`DEP-F9`) — must land with or after references | workspace | H | M |
| Build-once CI with a `dist` artifact (`CI-F1`) | CI | H | M |
| Parallel daemon tests; re-enable Cachix; run all package suites (`CI-F2`, `CI-F9`, `PKG-F15`) | CI | H | S |
| One table-driven workspace build + cache parity across lanes (`TLS-F1/F5/F2/F3/F4`) | tools/pack | H | M |
| Pin actions to SHAs (`CI-F11`) | CI | M | S |

### Wave 3 — Structural decomposition

Each of these is a multi-PR effort; land the matching test first so the refactor has a net.

| Item | Component | Prereq |
|---|---|---|
| Split `server.ts`: `chat-run/` service, then migrate inline routes (`DMN-F3`, `DMN-F4`) | daemon | registrar tests (`DMN-F10`) |
| Decompose `cli.ts` into `src/cli/<domain>.ts`, drop `@ts-nocheck` per file (`CLI-F4/F12/F13`) | daemon | parity spec (`CLI-F14`) |
| Split `FileViewer.tsx` / `HtmlViewer` (`WEB-F1`) | web | — |
| App-catalog store, one catalog at a time (`WEB-F2`) | web | `WEB-F4` helpers |
| Shared stream-parser core + streamFormat table (`DMN-F5`, `DMN-F7`) | daemon | golden mocks (`DMN-F11`) |
| Split `tools/pack/src/linux.ts` into `linux/` + shared lifecycle (`TLS-F10`) | tools/pack | `TLS-F1`, dispatch test |
| Contracts pruning + move `prompts/` to daemon (`PKG-F1`, `PKG-F5`) | packages | purity + usage tests |
| Global CSS triage → orphan guard → CSS Module migration (`WEB-C7/C9/C15`) | web | expanded-CSS helper extension |

### Wave 4 — Durable guardrails

Without these, every earlier wave decays.

| Guard | Prevents |
|---|---|
| `scripts/lint-content.ts` (`CNT-F12`) | phantom craft slugs, broken side-files, unbudgeted skill bodies |
| Byte budget in `composeSystemPrompt` (`DSY-F2`) | token-cost regressions |
| CLI↔API parity spec in `e2e/tests/` (`CLI-F14`) | single-surface capabilities |
| Contracts purity + export-usage tests (`PKG-F4`, `PKG-F1`) | boundary drift, dead DTOs |
| Markdown link checker + no-absolute-paths rule (`DOC-F12`, `DOC-F11`) | doc rot |
| Orphan-CSS + legacy-primitive ratchets (`WEB-C9/C15`) | CSS regrowth |
| `eslint-plugin-react-hooks`, package-scoped (`WEB-F10`) | stale-closure bugs |
| `onlyBuiltDependencies` assertion (`DEP-F12`) | supply-chain drift |
| Offline-safety guard on design-system previews (`DSY` risk note) | CDN URLs re-entering previews |

---

## 5. Invariants that constrain every change

These are not preferences; violating them breaks shipping behavior.

1. **Sidecar stamps have exactly five fields** — `app, mode, namespace, ipc, source`, owned by `packages/sidecar-proto/src/index.ts:60`. No orchestration layer may hand-build `--od-stamp-*` args or process-scan regexes.
2. **Channel identity is public and distinct** — `Open Design` / `Open Design Beta` / `Open Design Preview`; Windows beta validation must use the real `release-beta-win` namespace. Tightening mac's matcher (Wave 0) is a deliberate behavior change for ad-hoc namespaces and must be called out in the PR.
3. **stream-json stdin lifecycle** — stdin closes only when `pendingHostAnswers` is empty *and* a `turn_end`/`usage` arrives with a non-`tool_use` stop reason; `claude-stream.ts` emits `turn_end` *after* iterating content blocks. No parser consolidation may reorder or batch across one assistant wrapper.
4. **Contracts purity** — no Next/Express/Node fs-process/browser/SQLite/daemon/sidecar dependencies; moves go outward, never inward.
5. **Dual-track capability** — endpoint + UI + `od` subcommand land in the same PR, all on the same `/api/*` shapes.
6. **Data-dir precedence** — `OD_MEDIA_CONFIG_DIR > OD_DATA_DIR > <projectRoot>/.od`; runtime paths are namespace-scoped and never encode a port. A persisted catalog index lives under the data dir, since `OD_RESOURCE_ROOT` is read-only when packaged.
7. **Cascade semantics** — `index.css` is import-only and its order is intentional; CSS refactors must verify the expanded content and order, and dedupe must preserve the *last* definition's values.
8. **Typed `Dict` stays exhaustive** — lazy locale loading must use typed loaders, not `any`.
9. **Skills-protocol compatibility** — a plain Claude Code `SKILL.md` works with zero config; content lint applies to committed content only and never rejects user-imported skills at runtime.
10. **Design-system contract** — `DESIGN.md` + `tokens.css` + `components.html` are 150/150 and load-bearing; a prompt budget must never truncate the `tokens.css` block, and clone systems should be aliased, not deleted (saved user selections).
11. **Root command boundary** — no root `build`/`test`/`dev` aliases; new scripts stay package- or tool-scoped.
12. **Lockfile → Nix hash** — every `pnpm-lock.yaml` change needs `pnpm nix:update-hash` and `nix flake check`.
13. **Test placement** — `tests/` sibling to `src/`; Playwright in `e2e/ui/`; cross-app consistency checks in `e2e/tests/`.
14. **Mock PATH overlay** — all 18 bin names must keep resolving; the 4–9-line dispatchers are already minimal and should not be "consolidated".

---

## 6. Validation plan

- **Every PR:** `pnpm guard` + `pnpm typecheck` plus the package-scoped tests/builds matching the touched files.
- **Token-cost work (Waves 1, 4):** measure injected bytes per turn before and after for a fixed matrix (default / p90 / `stripe` design system × a craft-heavy template) and record it in the PR body. `docs/design-systems-token-audit.md` is the baseline.
- **Hot-path work (`CLI-F1/F2`, `DMN-F1`):** land a benchmark-shaped assertion (fs-ops per request; DB writes per 100 deltas) that goes **red on `main`** first, per the bug follow-up workflow.
- **Stream/parser work:** replay through `mocks/` with recordings fetched — `mocks-golden.test.ts` passes vacuously when they are absent, so verify it actually ran.
- **Packaging work:** `pnpm tools-pack mac build --to all` plus the identity tests; for the channel fix, add a `beta-local-flow` → *not* `Open Design Beta` case to `mac-identity.test.ts`.
- **CSS work:** the expanded-import helper must show identical content and order; `tests/styles/*` assert literal token strings, so tokenizing values requires updating those specs in the same PR.
- **Desktop/UI work:** stand up a buggy-vs-fix comparison across two namespaces where the symptom needs an eye; seed only through production HTTP APIs.

---

## 7. Deliberately not proposed

- **Deleting the 22 clone design systems.** Saved user selections would break; the token audit's `aliasOf` route is the safe one, and `components.html` is 150/150 distinct, which argues for re-tuning tokens rather than removing systems.
- **A virtualization library for the chat log.** `ChatPane` already hand-rolls it, and the `AssistantMessage` memo comparator plus `assistantCallbacksRef` are what keep streaming from re-rendering every message.
- **Consolidating the mock bin scripts.** 115 lines total against 1,217 shared; the PATH overlay depends on the names existing.
- **Adopting Tailwind in `apps/web`.** It is installed and running with zero directives; pick removal, not adoption, unless someone wants it deliberately.
- **Restoring a maintainer PR-duty tool.** Out of scope by explicit prior decision.
- **Changing the landing page's `0.18s ease` motion.** It follows its own DESIGN.md §7; record the exemption in AGENTS.md rather than "fixing" it toward the app's easing rule.

---

## Appendix — measurement provenance

Twelve read-only audits, run in parallel on 2026-09-03/04 against branch `docs/design-resource-reference` (HEAD `fbf826ef`), each scoped to one component and its `AGENTS.md`. Counts come from `wc`/`grep`/`find`/`du` sweeps, front-matter parsing, lockfile graph walks, and registry `dist.unpackedSize` metadata (`node_modules/` was not installed, so install sizes are registry-derived and understate `electron`, which downloads ~250 MB post-install). Per-package audit reports, including findings not promoted into this document, are the source for every `path:line` reference above.

Known measurement gaps, stated rather than guessed: `apps/landing-page`'s inlined-CSS byte totals and page count need one real `build:static` run to confirm; the repo is a shallow checkout, so per-file `git log` churn is not evidence and cron schedules were used instead.
