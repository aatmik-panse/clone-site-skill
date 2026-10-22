# Assembly: review the team's work, wire it, gate it, close the gap

Step 5. Input: N finished section files plus their `report.md` / `requests.json`. Output: a wired, built,
measured clone and `.clone/CLONE-REPORT.md`. Every number here is a gate. Never claim a gate you did not run.
`cdp:` = `mcp__plugin_chrome-devtools-mcp_chrome-devtools__` throughout.

Preconditions: `run.json.phase == "assemble"`; every section is `done`, `failed`, or `skipped`; the original's
per-section captures exist (`orig-w<width>.webp` for agents, `orig-w<width>.png` for the diff — §4).

## Review order

Cheapest first. Never screenshot what a grep would have caught.

| # | Pass | Browser? | Catches |
|---|---|---|---|
| 1 | Static sweep per section | no | ~40% of defects: token drift, paraphrased copy, forbidden imports, missing file |
| 2 | Drain `requests.json` | no | cross-section conflicts, missing tokens, dependency creep |
| 3 | Wire + build | no | import errors, type errors, wrong export names |
| 4 | Capture the clone | yes | — |
| 5 | Diff + geometry + gates → `VERIFY.md` | yes | everything measurable |
| 6 | Read `report.md`, **only** for sections that failed a gate or flagged a deviation | no | why the agent did what it did |

Reading 20 passing reports is waste. Reading a failing agent's report *before* its geometry table is also
waste — numbers say where to look, the report says why.

## 1. Static sweep

Spawn one `clone-reviewer` per finished wave (no browser, safe to run alongside anything). Give it the wave's
`outFile` list, each `spec.json`, and this table. It returns verdict rows; you fill `run.json.sections[].gates`.

| Check | Command / method | Gate |
|---|---|---|
| File exists, non-trivial | `wc -c <outFile>` | > 200 bytes |
| Export matches `spec.exportName` | `grep -c "export function <Name>"` | ≥ 1, and no `export default` |
| Hardcoded colors | `grep -Eoc '#[0-9a-fA-F]{3,8}\b\|rgba?\(\|hsla?\(\|oklch\(\|oklab\(' <outFile>` | **0** |
| Arbitrary values | `grep -Eoc '\-\[[0-9.]+(px\|rem\|em\|%)\]' <outFile>` | ≤ **3**, each justified in `report.md` |
| Forbidden imports | every `^import … from '(.+)'` against `spec.dependencies.allowed` | 0 outside the allowlist |
| Text fidelity | normalize both (collapse whitespace, strip markup), Levenshtein ratio vs `content.md` | ≥ **0.995** |
| Asset paths | every `src=` / `url()` appears in `spec.assets[].ref` (a public path — a bare `img/…` relative src is a fail, not a pass) | 100% |
| `"use client"` | present only if `report.md` states why | justified or absent |
| Effects present | for each `spec.effects[]` row, its `prop` appears in the file (`backdrop-blur`/`backdrop-filter`, `mix-blend-*`, `clip-path`, `mask-image`, `filter`/`blur-*`) with the measured `value` | **100%** — no probe and no pixel score reliably catches a missing glass blur or blend mode |
| Form fidelity | for each `spec.form.fields[]`: `type`, `name`, `required`, `autocomplete`, `inputmode`, verbatim `placeholder` present; `appearance` preserved where measured; `::placeholder` styled; every `states[]` row expressed | **100%**, and arbitrary values inside these rows do not count against the ≤3 budget |
| Spec compliance | every `spec` measurement (padding, gap, size, weight, radius, max-width) present as a token utility | no silent omissions |

A section failing only static checks is repaired without a screenshot. Fix trivia yourself — stray
`"use client"`, wrong import path, missing `aria-labelledby` — instead of re-spawning an agent for one line.

## 2. Drain requests

Read every `.clone/sections/*/requests.json`. You own every shared file, so every request is applied centrally
and recorded in `run.json.requests.applied[]` or `.rejected[]`.

| Kind | Action |
|---|---|
| `token` | Verify the value exists in `foundation.json` (`color.palette`, `type.scale`, `space.scale`, `patterns.*`). Measured → add under the §G role vocabulary. Within ΔE < 2 of an existing token → reject, name the existing one. Absent from `foundation.json` → reject as invented. |
| `dependency` | Approve only when no CSS-native equivalent exists (`scroll-snap`, `animation-timeline: view()`, `@starting-style`, `popover`). Default answer is no. Install once yourself. |
| `shared-component` | Requested by ≥2 sections → build it in the shared dir and patch both importers yourself. By 1 → stays local. |
| `global-css` | `@keyframes`, `@property`, `@supports`, custom-property fallbacks → accept into the token/global file. Layout or component rules → reject, they belong in the component. |

Conflicts, resolved by you and never by write-order:

| Conflict | Resolution |
|---|---|
| Same token name, two values | Keep the one matching `foundation.json`. Both real (two distinct measured surfaces) → add both with distinct role names. |
| Two agents hardcoded the same color | Real measured value → add token, patch both. Invented → force the nearest token. |
| Two sections built the same sub-component | Extract to the shared dir, edit both imports. Neither agent does this. |
| Sections overlap in the DOM (segmentation error) | Your call: re-cut the boundary and re-run both, or merge into one section and one file. Never ship duplicated markup. |
| `"use client"` on a static section | Remove it yourself. No re-spawn. |
| Two agents wrote the same asset filename | Impossible by construction — agents never write the asset dir. |

## 3. Wire and build

1. Import every `done` section into `run.json.stack.pageFile` in `sections.json` order (`order` ascending);
   `role: header|footer|sticky-cta` mount in the layout, not the page.
2. Every `skipped` section becomes one comment at its slot: `{/* 05-pricing — skipped by --sections */}`
   (`--static`: `<!-- 05-pricing — skipped by --sections -->`).
3. `--static`: concatenate `sections/<id>.html` into `index.html` in that order, keeping every `data-section`.
4. Build with the command in `run.json.stack` (per-mode commands: `references/stacks.md`). Attribute a build
   failure to a section by the error's file path and repair it **before** any screenshotting — a broken build
   makes every downstream gate meaningless.

### 3.1 Assembly defects you will actually hit

These are what break a fan-out build. Every one is a one-line orchestrator edit — fix it yourself, do not
re-spawn an agent for a renamed attribute. Sweep all of them in one grep pass over the wave's `outFile` list
before you run the build.

| Defect | Detect | Fix |
|---|---|---|
| HTML attributes left in JSX | `grep -nE '\b(class\|for\|stroke-width\|stroke-linecap\|stroke-linejoin\|stroke-dasharray\|clip-path\|clip-rule\|fill-rule\|fill-opacity\|stop-color\|stop-opacity\|xlink:href\|xmlns:xlink)=' <outFile>` | `className`, `htmlFor`, `strokeWidth`, `strokeLinecap`, `strokeLinejoin`, `strokeDasharray`, `clipPath`, `clipRule`, `fillRule`, `fillOpacity`, `stopColor`, `stopOpacity`, `href` (drop the `xlink:` prefix entirely) |
| Unclosed void elements | `grep -nE '<(img\|br\|hr\|input\|source\|path\|circle\|rect\|use\|stop\|line\|polyline\|meta\|link)\b[^>]*[^/]>'` | self-close: `<img … />`. Pasted SVG markup is the usual source. |
| Unescaped text in JSX | build error `Unexpected token`, or `react/no-unescaped-entities`; stray `{` `}` `<` `>` in copy | wrap the run in an expression — `{"Don't guess — measure"}` — and keep the glyph byte-identical to `content.md`. Swapping `’` for `'` to dodge the error is a text-fidelity failure. |
| `"use client"` below line 1 | Next: *the "use client" directive must be put at the top of the file* | move it above every import, or delete it — a section with no hook, no event handler and no browser API does not need it |
| Duplicate export names | `grep -rhoE 'export function [A-Za-z0-9_]+' <sectionsDir> \| sort \| uniq -d` | rename the later collision per the component-naming rule and patch the page import (both are yours) |
| Unused vars / implicit `any` | `tsc --noEmit` under strict mode | delete the dead binding; type the prop. Never widen to `any` and never disable the rule. |
| Missing React `key` in `.map` | console warning on the clone → fails the console gate | `key={item.id ?? i}` on the mapped root element |
| Section renders at ≈0 height | clone manifest `box.h < 8` while the original is hundreds of px | the reveal ships as `opacity:0` with no visible default, or the data array is empty. Content must be visible by default and the reveal enhance it (`motion.json.bySection.<id>.reducedMotion.clonePlan`). |
| One section overflows horizontally | global overflow probe reports `docOverflow: true` at one width, offenders all inside one section | fix that section (a fixed `w-[…]`, a non-wrapping grid, an unclipped absolute child). Never `overflow-x: hidden` on `body` — it hides the bug and kills `position: sticky`. |
| Wrong import path for the mode | build cannot resolve `@/components/…` in `--static` | use the path map in `run.json.stack`; `--static` sections are HTML partials with no imports at all |

## 4. Capture the clone

Serve the built clone (command per mode: `references/stacks.md`), then per gated width:

```
cdp:new_page        { url: "http://localhost:3000/" }          # once; reuse the pageId
cdp:resize_page     { width: <w>, height: 900 }
cdp:emulate         { viewport: "390x844x3,mobile,touch", userAgent: "<the UA used for the original>" }
cdp:evaluate_script { function: "<entire contents of scripts/extract-sections.js>",   # installer
                      filePath: ".clone/raw/sections-clone-<w>.json" }
cdp:evaluate_script { function: "() => window.__clone.sections.all()",                # overwrites with data
                      filePath: ".clone/raw/sections-clone-<w>.json" }
```

Then per section, anchor it and shoot both formats back to back from the same viewport state:

```
cdp:resize_page     { width: <w>, height: <min(section.box.h, 4000)> }
cdp:evaluate_script { function: "() => { document.querySelector('[data-section=\"<id>\"], #<id>')
                        .scrollIntoView({block:'start',behavior:'instant'}); return {y: scrollY}; }" }
cdp:take_screenshot { format: "webp", quality: 88, filePath: ".clone/sections/<id>/clone-w<w>.webp" }
cdp:take_screenshot { format: "png",               filePath: ".clone/sections/<id>/clone-w<w>.png" }
```

- **Same width, same scroll anchor, same device flags, same `resize_page` height as the original's capture** —
  otherwise the diff measures your capture, not the clone. `emulate` resets every omitted option: restate it.
- `.webp` is what a repair agent reads; `.png` is what `visual-diff.mjs` decodes (no WebP decode without npm).
  Sections taller than 4000 px: one `fullPage: true` shot at that width, both sides.
- `orig-w<w>.png` is written back in step 4 alongside the webp (`references/sectioning.md` §7), so it should
  already be on disk. If it is not — a run that predates that pairing, or a section captured through the
  `uid` route — that is an anomaly worth a `run.json.notes` line, and the recovery is: re-navigate to the
  original, prewarm, re-shoot **only the sections you are about to gate**. Never re-measure the whole
  original for a diff, and never substitute `--loose` for the missing PNG.
- Every screenshot and every `evaluate_script` passes `filePath`. No image ever attaches into context.

Then, once per gated width, the page-level pair that feeds §5.3's full-page gate — from the top of the page,
`scrollY === 0`, after a full prewarm scroll so lazy content has painted:

```
cdp:take_screenshot { fullPage: true, format: "webp", quality: 88, filePath: ".clone/screenshots/clone-w<w>-full.webp" }
cdp:take_screenshot { fullPage: true, format: "png",               filePath: ".clone/screenshots/clone-w<w>-full.png" }
```

The original's counterpart (`orig-w<w>-full.webp` / `.png`) was shot in step 4 by `references/sectioning.md` §7.
Under `--profile cheap` neither pair is taken and the full-page gate is reported as `not-run`, never as passed.

## 4b. Capture determinism — do this or the diff measures the clock

On an animated site a naive capture compares two different *moments*, not two designs. Every one
of these cost a real run a wrong conclusion before it was fixed, and the signature is always the
same: **low `aHash` (structure matches) with `worstTile` near 1.0 (one region completely
different)**.

Freeze identically on **both** sides, immediately before every screenshot:

```js
// 1. kill CSS animation and transition
await page.addStyleTag({ content: `*, *::before, *::after {
  animation: none !important; transition: none !important;
  scroll-behavior: auto !important; caret-color: transparent !important; }` });

// 2. pin every video to frame 0 — a <video> frame is a function of elapsed time, so two
//    independent page loads never agree on it. Re-run after EVERY resize: a resize can
//    restart autoplay.
await page.evaluate(async () => { for (const v of document.querySelectorAll('video')) {
  try { v.autoplay = false; v.pause(); if (v.readyState >= 1) v.currentTime = 0; v.pause(); } catch {} } });

// 3. clear JS-written INLINE transforms (scroll parallax). A CSS `animation: none` cannot
//    reach them. Inline only: transforms authored in classes are legitimate layout.
await page.evaluate(() => { for (const el of document.querySelectorAll('[style]')) {
  const st = el.style; if (st.transform || st.translate || st.scale || st.rotate) {
    st.transform = ''; st.translate = ''; st.scale = ''; st.rotate = ''; } } });
```

Measured payoff on one run: freezing the hero video alone took that section from **0.943 → 0.018**.

| trap | symptom | fix |
|---|---|---|
| `<video>` frame differs | whole-section diff ~0.9 with content present on both sides | pin `currentTime = 0`, re-pin per width |
| marquee/slider phase | a band of the section is shifted horizontally | freeze animation; both sides then sit at the same base offset |
| JS inline transforms | one tile at 1.0, `aHash` 2–6 | clear inline transform/translate/scale/rotate |
| consent banner asymmetry | two unrelated sections report **byte-identical** numbers | dismiss on **both** sides (seed the clone's own storage key), and verify the banner in its own pass, undismissed on both |
| a stale dev/prod server | *every* diff ≈ 1.0, page renders unstyled | own the server lifetime, and abort if a known token (e.g. body background) is absent |

**Do not "fix" this by matching scroll position.** Capturing each section with its own top at the
viewport top, tiled, on both sides, is the obvious idea and it was tried: mean diff got **worse**
(0.217 → 0.278 at 1440). Their JS re-applies transforms after each resize, so the extra scrolling
buys nothing and disturbs more than it settles.

## 5. The gate suite

### 5.1 Per section, per gated width — desktop = width ≥ 1024, narrow = width < 1024

| Gate | `run.json` field | Method | PASS | WARN | FAIL |
|---|---|---|---|---|---|
| Pixel diff, desktop | `gates.byWidth.<w>.pixelDiff` | §6 | ≤ **0.025** | 0.025–0.060 | > **0.060** |
| Pixel diff, narrow | same | §6 | ≤ **0.040** | 0.040–0.090 | > **0.090** |
| Perceptual hash | `gates.byWidth.<w>.aHash` | 8×8 gray aHash, Hamming/64 | ≤ **5** | 6–10 | > **10** |
| Worst tile | `gates.byWidth.<w>.worstTile` | 8 cols × 12 rows over the section | ≤ **0.20** | 0.20–0.35 | > 0.35 |
| Horizontal overflow | `gates.byWidth.<w>.overflowOffenders` | `() => window.__clone.responsive.overflow()` (`references/responsive.md`) | **0** | — | ≥ 1 |
| Geometry probes | `gates.geometryPass` / `geometryWorstPx` | §5.2 | ≥ **0.90** and no probe with \|Δy\| > 16px | 0.85–0.90 | < 0.85 |
| Text fidelity | `gates.textFidelity` | §1 | ≥ **0.995** | 0.98–0.995 | < 0.98 |
| Token drift | `gates.hardcodedColors` | §1 grep | **0** | — | ≥ 1 |
| Arbitrary values | `gates.arbitraryValues` | §1 grep | ≤ **3** | 4–8 | > 8 |
| Console | `gates.consoleErrors` | `cdp:list_console_messages {types:["error"]}`, attributable to this section | **0** | — | ≥ 1 |
| Section height | — | `diff.json.sizeDelta.hPct` | ≤ **2%** | 2–5% | > 5% |

`verdict = "pass"` requires every row at PASS; any FAIL → `"fail"`; otherwise `"warn"`. A `warn` does not block
`done` but must appear in `CLONE-REPORT.md` with its numbers.

### 5.2 Geometry probes

Re-run `scripts/extract-sections.js` on the clone (§4) and join `probes[]` **by `name`**, never by selector —
the clone's DOM has different `nth-of-type` paths, and `name` (`root`, `heading`, `body`, `cta`, `media`,
`eyebrow`, `repeat-item`) is the closed vocabulary that makes the join legal.

| Property | Tolerance | Property | Tolerance |
|---|---|---|---|
| `box.y` (section-relative top) | `max(4px, 1.5% of section height)` | `font.letterSpacing` | ≤ **0.01em** |
| `box.h` | `max(4px, 2% of value)` | `color`, `bg` | exact after `rgb()` normalization |
| `box.w` | `max(4px, 1% of value)` | `radius` | ≤ **1px** |
| `font.size` | ≤ **0.5px** | `pad` (each side) | ≤ **2px** |
| `font.weight` | exact | `shadow` | offsets ≤ 1px, alpha Δ ≤ 0.02 |
| `font.lineHeight` | ≤ **1px** | | |

`geometryPass` = passing rows / total rows. Write `.clone/sections/<id>/geometry.md` with fixed columns and the
cause in a trailing comment — this is the file a repair agent reads:

```
probe        prop         original         clone            Δ         tol      verdict
root         box.h        968px            981px            +13px     19px     PASS
heading      box.y        128px            146px            +18px     14px     FAIL  <- padding-block wrong
repeat-item  box.w        365.33px         352px            -13.33px  4px      FAIL  <- gap 32 vs 40
```

### 5.3 Global gates — once, before `phase: "done"`

| Gate | Method | Threshold | `run.json` field |
|---|---|---|---|
| Font files present | every `foundation.fontFamilies[].files` entry exists on disk; every `@font-face` `src` is a local path | **100%** | — |
| No runtime font CDN | `cdp:list_network_requests {resourceTypes:["font","stylesheet"]}` → zero hits on `fonts.googleapis.com`, `fonts.gstatic.com`, `use.typekit.net`, `p.typekit.net` | **0** | — |
| Console clean | `cdp:list_console_messages {types:["error"]}` after a full prewarm scroll | **0** errors, ≤ 2 warnings | `verify.overall.consoleErrors` |
| No 4xx/5xx | `cdp:list_network_requests` → any `status >= 400` | **0** | `verify.overall.http404` |
| Token drift, repo-wide | §1 color grep across sections + shared dirs, excluding the token file and `*.svg` | **0** | `verify.overall.tokenDrift` |
| Overflow sweep | overflow probe at **360, 390, 768, 1024, 1280, 1440, 1920** (`cheap`: gated widths only) | **0** offenders at every width | `verify.overall.overflowBreakpoints` |
| Full-page diff | `visual-diff.mjs .clone/screenshots/orig-w<w>-full.png .clone/screenshots/clone-w<w>-full.png` at the primary width (`standard`), every gated width (`thorough`), not run on `cheap` | mean ≤ **0.030**, no section at FAIL | `verify.overall.diffMean` / `diffWorst` |
| Document height | `\|Δ docHeight\| / original` at the primary width | ≤ **4%** | `verify.overall.docHeightDelta` |
| Build | the build command from `run.json.stack` | exit 0, zero type errors | — |
| Provenance complete | every mirrored file in the project asset dir appears in `.clone/PROVENANCE.md` with url + content-type + bytes + sha256 | **100%** | — |
| Fingerprint diff | §5.4 | ≥ **0.90** of rows in tolerance | — |
| a11y floor (`thorough`) | `cdp:lighthouse_audit {mode:"navigation", device:"desktop"}`, both sides | clone ≥ original − 2 | — |

### 5.4 Fingerprint diff

`foundation.json.fingerprint[]` is the numeric verification contract. Re-run `scripts/extract-foundation.js`
on the clone (`filePath: ".clone/raw/foundation-clone-<w>.json"`), join rows by `(tag, textHead)` — text is
verbatim, so it is a stable key — and compare `fontSizePx`, `fontWeight`, `lineHeightPx`, `resolvedFamily`,
`color`, `bg`, `textWidthPx` with the §5.2 tolerances (`textWidthPx` ≤ 1.5%, the real test of whether the right
font file loaded). A `resolvedFamily` mismatch means fallback rendering: FAIL the font gate, not a geometry warn.

### 5.5 Motion and responsive parity

Static, cheap, and invisible to a pixel diff — which is exactly why it needs its own gate.

| Check | Method | Gate |
|---|---|---|
| Keyframes present | every `motion.json.cssom.keyframes[].name` used by a shipped section exists in the clone's CSSOM | 100% |
| Durations / easings | per `bySection.entrance[]`: clone duration within **±20%**, same easing family | ≥ 90% of entries |
| Reduced motion | `motion.json.reducedMotion.authored == true` → clone has a `@media (prefers-reduced-motion: reduce)` block over those selectors | present |
| Evidence rule | an entry that reached a shipped file without `evidence` → drop the behaviour | 0 unsourced |
| Breakpoints | `responsive.json.breakpointSystem.boundaries` vs the clone's authored media queries; `reorder[]` and `breakpointOnly[]` reproduced at the measured widths | same set, 100% |
| `clamp()` fidelity | sample the clone at each `deltas[].fluid` fit's min and max width | within **1px** |

`:active` and `prefers-reduced-motion` are not verifiable through this MCP. Check them by reading the emitted
CSS and mark them `authored-not-verified` in the report.

### 5.6 `VERIFY.md`

Rewrite `.clone/VERIFY.md` every verify pass, worst-first, and update
`run.json.verify.{lastRunAt,overall,failing,history}` in the same transition.

```md
# VERIFY — iteration 1 · 2026-07-30T19:04:11Z
overall: diffMean 0.024 · diffWorst 0.058 · gates 21/23 · console 0 · 404s 0 · tokenDrift 0 · docHeightΔ 1.8%

| section | verdict | w1440 diff | w390 diff | aHash | worstTile | geom | text | notes |
|---|---|---|---|---|---|---|---|---|
| 06-testimonials | FAIL | 0.071 | 0.088 | 9 | 0.34 @ x480 y320 | 0.72 | 1.000 | card gap 40 vs 32 |
| 03-features | warn | 0.031 | 0.037 | 6 | 0.22 | 0.91 | 0.998 | icon box 2px |
```

## 5b. Five ways a gate lies, and what to gate instead

Each of these produced a phantom failure on a real run. The pattern to internalise: **when a
whole class of sections fails identically, suspect the instrument before the code.**

| the lie | why | gate instead |
|---|---|---|
| text fidelity ~0.22 on *every* section | the wanted-strings list was scraped from `content.md` including `selector`/`key`/`role` rows, so CSS paths were being demanded as visible copy | extract only `.text`/`.alt` rows; compare with **all whitespace removed** (`textContent` concatenates adjacent elements with no separator); an **`alt` attribute satisfies an alt string** — it can never appear in `textContent`, so image-only sections otherwise score ~0.02 no matter how correct they are |
| geometry "misses" of 90–200 px | probes matched **first-of-tag independently on each side**; in a frozen marquee "first `img`" is arbitrary | skip a probe when the origin box does not intersect the section root, or when the section holds >3 of that tag; a tag **absent** on the clone is a *structural substitution* (`<summary>` for an Alpine `<button>`), not a misplacement |
| a section 3.6× too tall scoring 0.166 | the differ compares only the **overlapping region**, so height error is invisible | surface `sizeDelta.h` as `dh` and gate it (`|dh| > 4` fails). This is what exposed a missing `margin` that a **0.0000** pixel diff had completely hidden |
| half the residual on text-heavy sections | clips are rounded independently on each side, so a section lands on a different sub-pixel row and every glyph anti-aliases differently | search `dy ∈ {-1,0,+1}`, report `pixelDiffAligned`, gate on that, keep the strict value alongside |
| `null` failing a gate that never ran | `null < 0.98` coerces to `0 < 0.98` | guard explicitly: a section with no comparable strings has no text gate |

Two of those changes make the suite **stricter**, not looser — height gating and the alignment
search — and the height gate is the one that found a real defect. If a metric change only ever
turns failures into passes, be suspicious of it.

## 6. Running the diff

```bash
node ~/.claude/skills/clone-site/scripts/visual-diff.mjs \
  .clone/sections/06-testimonials/orig-w1440.png \
  .clone/sections/06-testimonials/clone-w1440.png \
  .clone/sections/06-testimonials/diff.json \
  --threshold 12 --cols 8 --rows 12 --out .clone/sections/06-testimonials/diff-w1440.png
```

`--threshold` = per-channel delta (12/255) that counts a pixel as different · `--cols`/`--rows` = heat grid ·
`--gate R` = exit 1 when `pixelDiff > R`, the cheapest way to branch in Bash · `--json` = report to stdout ·
`--out` = red-on-gray diff map. Both images are 2× box-downscaled first, which is what makes the score
insensitive to antialiasing and subpixel text.

Read from `diff.json`: `pixelDiff`, `aHash`, `worstTile`, `worstTileBox`, `worstTiles[0..4]` (top five regions,
in **original** pixel coordinates), `sizeDelta`. Never re-print `tiles[]` into context — 96 numbers a repair
agent can read from disk. `worstTiles` is the point of the exercise: "18% mismatch at x 480–660, y 320–400"
tells a repair agent where to look; a scalar tells it nothing. Paste those boxes into the repair prompt verbatim.
A size mismatch is a finding, not a nuisance: the score covers the overlapping region, `sizeDelta.hPct` is
reported separately, and neither image is ever rescaled to match the other.

Degradation path, in order: (a) inputs must be PNG — re-shoot with `format:"png"` if you only have WebP;
(b) `--loose` returns a byte/size heuristic marked `degraded: true`, usable as a smoke test and never as a
fidelity number; (c) the in-browser alternative — with both images reachable over the clone's own dev server
(same origin, no canvas taint), `cdp:evaluate_script` can `createImageBitmap` → `OffscreenCanvas` →
`getImageData` and compute the identical fields; it reads WebP but emits no `worstTiles`. Record any use of
(b) or (c) in `run.json.notes`.

## 7. The bounded repair loop

```
K    = run.json.budget.repairMax           # set per profile in references/scaling.md
Amax = run.json.budget.sectionAttemptsMax  # same source
iter = run.json.budget.repairUsed
loop:
  failing = sections where gates.verdict == "fail"
  if failing empty:                        break   # success
  if iter >= K:                            break   # budget exhausted
  if iter >= 1 and (prevDiffMean - diffMean) < 0.005
     and no gate flipped to pass:          break   # plateau — stop burning tokens
  if one failure signature covers >= 40% of failing:          # systemic
      fix it yourself in the orchestrator-owned file (tokens, container width, base color,
      type scale, layout shell). Do NOT spawn repair agents. Re-verify all sections.
  else:
      for each failing section with attempts < Amax: state = "needs-repair";
      spawn one clone-section agent each, min(len(failing), maxParallel) at a time
  iter++; repairUsed = iter; re-verify only what you touched; continue
```

Repair prompt = the section's original `PROMPT.md` plus the repair addendum defined in
`references/agent-brief.md`, carrying the failing gate values with their thresholds, the `worstTiles` boxes,
`geometry.md`, `diff.json`, `clone-w<w>.webp`, `orig-w<w>.webp`, and the instruction to fix **only** the failing
measurements — churn on passing parts regresses a good score.

Systemic signatures to recognize before spawning anything: every `box.w` short by the same amount (container
max-width), every `box.y` drifting cumulatively (section padding scale), every heading 2px off (type scale or
`line-height` rounding), every background subtly wrong (surface token), narrow widths failing while desktop
passes (breakpoint boundary). One edit fixes 20 sections; 20 agents fix it 20 inconsistent ways.

Plateau is measured on `verify.overall.diffMean` across iterations and recorded in `verify.history[]`: an
absolute improvement below **0.005** with no gate flipping to pass means the remaining error is not the kind
another agent turn removes. Stop there even if `iter < K`.

When a section will not converge — `attempts == Amax`, or an iteration that improved its own mean diff by less
than 0.005 — stop. Set `state: "failed"`, keep the best version on disk, write its row into `UNRESOLVED.md`,
move on. A clone at
`diffMean 0.034` with three documented gaps beats one claimed done at 0.034 with the gaps hidden.

## 8. `.clone/CLONE-REPORT.md`

Write once, at the end, from `run.json` — never from memory.

```md
# Clone report — <title> (<finalUrl>)
Run `<runId>` · captured `<capturedAt>` · profile `<profile>` · widths 1440, 390 · stack <mode/framework/css>

## Fidelity
diffMean **0.024** · worst **0.058** · gates **21/23** · console errors **0** · 404s **0** · token drift **0**
· docHeight Δ **1.8%** · overflow offenders **0** at 360/390/768/1024/1280/1440/1920

| section | file | verdict | w1440 | w390 | geom | text | note |
|---|---|---|---|---|---|---|---|
| 00-header | components/shared/SiteHeader.tsx | pass | 0.008 | 0.012 | 1.00 | 1.000 | shared, layout-mounted |
| 03-features | components/sections/FeaturesThatShip.tsx | warn | 0.031 | 0.037 | 0.91 | 0.998 | icon box 2px small |
| 06-testimonials | components/sections/Testimonials.tsx | fail | 0.071 | 0.088 | 0.72 | 1.000 | see UNRESOLVED.md |

## Mirrored
- Fonts: 4 files, 2 families, all local; no font CDN at runtime.
- Images/SVG/media: 63 files, 4.19 MB. Source URL, content-type, bytes, sha256 in `.clone/PROVENANCE.md`.
- Text: copied verbatim per section; text fidelity ≥ 0.995 everywhere.
- Failed downloads: 1 — `https://cdn…/hero.mp4` (403, hotlink-protected). Poster frame used instead.

## Not reproduced, and why
| Thing | Why | What shipped instead |
|---|---|---|
| Hero WebGL canvas | shader source not recoverable from the page | static poster PNG at the measured box |
| `:active` press states | not observable through this MCP | authored from CSSOM rules, `authored-not-verified` |
| Signup form submission | `spec.form.actionPolicy: "inert"` — a mirrored `action` would post a visitor's data to the original's owner | markup, labels and every state reproduced; `action` omitted and submit prevented. Measured endpoint recorded in the spec |

## Manual follow-ups
1. `components/sections/Testimonials.tsx` — card gap 40 vs measured 32 at ≥1024. One-line fix.
2. Re-run `--thorough` for the two overlay designs stubbed closed on this profile.

## Note
The mirror reproduces third-party text, marks, and imagery byte-for-byte; swapping or keeping them before you
publish is your call. Every mirrored file is listed in `.clone/PROVENANCE.md` with its source URL,
content-type, byte count, and sha256.
```

`.clone/UNRESOLVED.md` — written only when gates still fail after the budget. One row per residual gap: the
measured delta, the probable cause, the one-line manual fix.

```md
| section | gate | measured | target | probable cause | manual fix |
|---|---|---|---|---|---|
| 06-testimonials | w1440 pixelDiff | 0.071 | 0.025 | avatars render at 2× the measured box | set width/height 48 on the img |
| 06-testimonials | geometry box.w | -13.33px | ±4px | grid gap 40 vs measured 32 | `gap-8` instead of `gap-10` |
```
