# Design Systems — Token Fidelity Audit

**Status:** Findings v1 · 2026-08-21
**Parent:** [`spec.md`](spec.md) · **Siblings:** [`design-systems.md`](design-systems.md) · [`architecture.md`](architecture.md)
**Scope:** All 150 packages under `design-systems/`, audited across seven dimensions that the existing guards do not cover.

---

## Summary

The bundled catalog is structurally sound and semantically drifted.

Every package satisfies the token contract: **150/150 declare all 30 non-A2 tokens**, and
`design-tokens.json` agrees with `tokens.css` perfectly (mean colour overlap `1.000`, n=135).
Schema conformance is not the problem.

The problem is that nothing checks whether a system's `tokens.css` still describes the system
its `DESIGN.md` documents. Three findings follow from that gap:

| # | Finding | Scale |
|---|---|---|
| 1 | `atelier-zero` ships a stub `tokens.css` that contradicts its own `DESIGN.md` on every axis | 1 system, flagship |
| 2 | The 150 packages collapse to **128 distinct token files**; 7 clone groups are byte-identical | 22 redundant systems |
| 3 | One 15-system chunk of the 2.0 packaging backfill never landed | 15 systems |

All three are invisible to CI today. Section 6 explains why.

---

## 1. `atelier-zero` — stub tokens (severity: high)

`atelier-zero` is the flagship hand-authored system: 316 lines, 12 sections, the canonical
system behind `design-templates/open-design-landing/` and `open-design-landing-deck/`.

Its `DESIGN.md` documents 12 specific colours and a three-family type pairing.
**Zero of those 12 colours appear in `tokens.css`.**

| | `DESIGN.md` documents | `tokens.css` ships |
|---|---|---|
| Ground | Paper `#efe7d2` (warm ivory) | `#ffffff` |
| Ink | `#15140f` — *"pure black is forbidden"* | `#111111` |
| Accent | Coral `#ed6f5c` — *"one coral moment per ~600vh"* | `#111111` (collapsed onto `--fg`) |
| Display | `Inter Tight` 700–900 | `"Helvetica Neue", Arial, sans-serif` |
| Serif | `Playfair Display` Italic 500 | — |
| Mono | `JetBrains Mono` | `"SF Mono", ui-monospace` |

The stub propagated to the whole machine-readable layer — `design-tokens.json`,
`components.html`, `tailwind-v4.css`, and all three `preview/` pages carry the same generic
values. Only the prose in `DESIGN.md` still describes Atelier Zero.

**Why this is worse than a cosmetic bug.** `atelier-zero/USAGE.md` prescribes the read order,
and step 3 is *"Paste `tokens.css` into the first artifact `<style>` block before writing
component CSS."* An agent following the system's own documented procedure gets no paper,
no coral, and none of the type pairing.

**The correct values already exist in-repo.** `apps/landing-page/app/globals.css` carries the
full documented palette (`--paper: #efe7d2`, `--ink: #15140f`, `--coral: #ed6f5c`,
Playfair Display + Inter Tight + JetBrains Mono) — the landing page was built by hand against
`DESIGN.md`, not against the tokens. Remediation is mechanical: rebuild `tokens.css` from
`globals.css` + `DESIGN.md` §2–3, then regenerate the dependent artifacts.

`warm-editorial`, the other hand-authored starter, has a milder version of the same problem —
see finding 2.

---

## 2. Cross-system duplication — 150 packages, 128 distinct token files

Hashing each package's full 56-token `:root` block:

- **128** distinct token files across 150 packages
- **7** exact clone groups covering **29** systems — **22 redundant**
- **40** systems sit in a group that is ≥90% token-identical to another

### Exact clones (all 56 tokens byte-identical)

| n | Systems |
|---|---|
| 7 | `bento` `contemporary` `corporate` `flat` `perspective` `professional` `simple` |
| 6 | `artistic` `cafe` `elegant` `refined` `storytelling` **`warm-editorial`** |
| 5 | `colorful` `creative` `energetic` `friendly` `vibrant` |
| 3 | `clay` `claymorphism` `neumorphism` |
| 3 | `cosmic` `neon` `totality-festival` |
| 3 | `dramatic` `futuristic` `sleek` |
| 2 | `paper` `vintage` |

### Near-clones (≥90% identical, differing only in one or two font tokens)

`clay`/`claymorphism`/`neumorphism`/`skeumorphism` and `cosmic`/`fantasy`/`neon`/`totality-festival`
differ from each other in `--font-display` alone. `paper`/`spacious`/`vintage` likewise.

Two observations:

1. **`warm-editorial` is a hand-authored starter that is now byte-identical to five imported
   skill systems.** Like `atelier-zero`, its own `DESIGN.md` documents different values
   (`#faf7f2` / `#1c1a17` / `#c0512f`) than it ships (`#fbf6ee` / `#201914` / `#9b5b32`).
   Both starters appear to have been overwritten by batch presets.
2. Styles that are *conceptually* opposed — `neumorphism` (soft, extruded) and `clay`
   (matte, chunky); `cosmic` (sci-fi) and `fantasy` (ornamental) — are chromatically
   indistinguishable. A user switching between them in the Design System dropdown sees no change.

This does not mean 22 systems should be deleted. It means either the tokens should be
differentiated to match their distinct `DESIGN.md` prose, or the duplicates should be
declared aliases so the catalog count stops overstating the variety on offer.

---

## 3. Incomplete 2.0 packaging — a 15-system contiguous block

15 packages are missing the entire 2.0 layer: `design-tokens.json`, `tailwind-v4.css`,
`components.manifest.json`, `manifest.json`, `USAGE.md`, `preview/`, and `source/`.
They have only `DESIGN.md`, `tokens.css`, and `components.html`.

```
spacious  spotify  starbucks  storytelling  stripe  supabase  superhuman  tesla
tetris  theverge  together-ai  totality-festival  trading-terminal  uber  urdu
```

Sorted alphabetically, these occupy **indices 120–134 of 150 — a perfectly contiguous block**,
bounded by `spacex` (complete) and `vercel` (complete). Everything before and after is packaged.

That is not a content pattern; it is an interrupted job. The commit is
`b3b5bbec design-systems: backfill 2.0 package batch 01 (#3781)` — one chunk of batch 01
never landed. Re-running the backfill over this slice should close it.

---

## 4. Semantic drift between `DESIGN.md` and `tokens.css`

Measuring what fraction of the hex values named in each `DESIGN.md` actually appear in that
system's `tokens.css`:

| Bucket | Count | Reading |
|---|---|---|
| Boilerplate doc table | 51 | `DESIGN.md` colour table is scaffolding — **tokens are authoritative** |
| ≥67% match | 19 | Aligned |
| 34–66% match | 48 | Partially retuned |
| <34% match | 23 | Substantially retuned |
| 0% match | 9 | Fully diverged (incl. `atelier-zero`) |

**The 51 are not a defect.** They are the `awesome-design-skills` import, whose `DESIGN.md`
files share an identical semantic colour triple (`#16A34A` / `#D97706` / `#DC2626`). That set
is a *perfect subset* of the 57 systems that also share identical boilerplate guardrail text —
the same import batch. For these, `tokens.css` holds the real hand-tuned identity and the doc
table is generic filler. Worth normalising, but not a fidelity bug.

The distinction matters for remediation: for the 51, fix the docs; for `atelier-zero`, fix the tokens.

### Typography drift

Same measurement over font families named in prose vs. declared in `--font-*`:

- **39** systems: every named face is declared
- **42** systems: partial
- **51** systems: **no** named face reaches `tokens.css`
- **18** systems: no faces named in prose

`atelier-zero` is the worst case — `DESIGN.md` names Inter, Inter Tight, Playfair Display and
JetBrains Mono; `tokens.css` declares none of them.

---

## 5. Accessibility of the token pairs

Contrast computed per WCAG 2.1, compositing `rgba()` values over their own `--bg`.

| Pair | Role | n | Below 4.5:1 |
|---|---|---|---|
| `--fg` on `--bg` | Body text | 150 | **0** |
| `--accent-on` on `--accent` | Button labels | 150 | **39** (10 below 3.0:1) |
| `--muted` on `--bg` | Descriptive copy | 150 | 33 |
| `--meta` on `--bg` | Tertiary metadata | 131 | 67 |

Body text is flawless across all 150 — that axis is well tended.

**`--accent-on` / `--accent` is the finding that matters.** This pair is the label on a primary
button, so it is real body-sized UI text. Worst cases: `duolingo` 2.09:1, `wechat` 2.38:1, and
the six-system `colorful`/`creative`/`doodle`/`energetic`/`friendly`/`vibrant` clone group at
2.86:1 — one duplicated palette carrying one defect into six systems.

Two caveats, stated so the numbers are not over-read: `--meta` is documented as a
tertiary/disabled tier, where WCAG exempts disabled controls, so the 67 should be triaged
rather than treated as 67 violations. And systems using these pairs only at large-text sizes
face a 3:1 floor, not 4.5:1. `--border` contrast was measured and deliberately excluded —
hairline dividers are not UI-component boundaries under 1.4.11, and reporting them would have
produced a misleading 145/150 "failure" rate.

---

## 6. Why CI does not catch any of this

Three independent blind spots, each with a concrete mechanism.

**`source/token-contract.report.json` grades `tokens.css` against itself.**
All 135 reports score **100 / "excellent"**, and **none** sets `recommendRebuild`.
`atelier-zero` scores 100. The per-token justification explains why:

> `"Bundled tokens.css declares --bg; no upstream recrawl was performed for this backfill."`

The contract validates that tokens are *declared*, using the token file as its own source of
truth. Identity drift is undetectable by construction.

**`check-design-system-package-quality.ts` early-returns for unmigrated packages.**

```ts
const migrated = isMigratedPackage(input.manifest);
if (!migrated) {
  return { migrated: false, score: 0, checks, violations };  // violations is empty
}
```

The 15 packages from finding 3 have no `manifest.json`, so they are not migrated, so the gate
returns an empty violations array. They cannot fail the check that exists to catch them.

**No guard compares `tokens.css` to `DESIGN.md`, and none compares systems to each other.**
`check-tokens-fixture-sync.ts` verifies `tokens.css` ↔ fixture parity; `check-design-system-manifests.ts`
validates manifest shape. Both are intra-package. Nothing reads the prose, and nothing would
notice seven packages shipping the same file.

---

## 7. Proposed guards

Three checks, in severity order. Each is cheap and deterministic.

1. **`check-design-system-token-fidelity`** — parse hex values and font families from
   `DESIGN.md` §2–3, assert a minimum overlap with `tokens.css`. Allowlist the 51 known
   boilerplate-table systems until their docs are normalised. Catches finding 1.
2. **`check-design-system-uniqueness`** — hash each `:root` block, fail on exact collisions
   unless the manifest declares an explicit `aliasOf`. Catches finding 2.
3. **`check-design-system-contrast`** — assert `--accent-on` on `--accent` clears 4.5:1, and
   `--fg` on `--bg` clears 7:1. Catches finding 5's real cases without the `--meta` noise.

A fourth change is a one-liner: make the package-quality gate report unmigrated packages as a
violation rather than returning silently, so finding 3 cannot recur.

---

## Method

All figures reproduce from a clean checkout with no network access.

- **Token parsing** — comment-strip `tokens.css` *before* locating `:root`; several files quote
  the string `:root { … }` inside their header comment, and matching that block first yields a
  false "missing `--bg`" on 12 systems.
- **Contract** — the 56-token schema and its four layers are read from
  `packages/contracts/src/design-systems/token-schema.ts`. The 30 non-A2 tokens are treated as
  required; A2 tokens carry documented fallbacks.
- **Colour fidelity** — set overlap of 6-digit hex literals in `DESIGN.md` against the values in
  the `:root` block, comments excluded so rationale prose does not count as a declaration.
- **Contrast** — WCAG 2.1 relative luminance, with `rgba()` composited over `--bg`
  (`notion`, `starbucks` and `xiaohongshu` express `--fg` with alpha).
- **Duplication** — SHA-1 over the sorted token map for exact clones; pairwise key-value
  agreement over the union of keys for the ≥90% near-clone grouping.

### Corrections made during the audit

Two intermediate results were wrong and are recorded so the numbers are not re-derived incorrectly:

- An initial pass reported **12 systems missing `--bg`**. That was the header-comment parser bug
  above. The correct figure is **0**.
- An initial pass classified **9 systems as token stubs**. Eight of those (`hud`, `lamborghini`,
  `voltagent`, `mono`, `neumorphism`, `totality-festival`, `trading-terminal`, `urdu`) are merely
  *retuned* — same colour family, different values. Only `atelier-zero` collapses to generic
  defaults (accent equal to `--fg`, white ground, default face). The correct figure is **1**.
