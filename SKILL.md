---
name: clone-site
description: Clones a website with high fidelity from its live source of truth. Use when the user says clone this site, copy this page, rebuild this landing page, recreate this design, make me a site like X, or pastes a URL and asks for the same thing in their stack. Drives Chrome DevTools MCP to measure computed styles, CSSOM, fonts, geometry, motion, and breakpoints, mirrors every asset byte-for-byte, recovers the motion system from the site's own bundle when it is hand-rolled, then builds it with a parallel team of section agents and verifies the result against the original.
user-invocable: true
argument-hint: "<url> [--static] [--pages N] [--sections ids] [--max-parallel N] [--viewport WxH] [--profile cheap|standard|thorough] [--cheap] [--thorough] [--out DIR] [--depth N] [--resume] [--refresh]"
license: MIT
metadata:
  author: aatmik
  version: "1.2.0"
  category: frontend
allowed-tools:
  - Task
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - mcp__plugin_chrome-devtools-mcp_chrome-devtools__*
---

Rebuilds a target URL in real code by reading the running page: computed styles, CSSOM rules and keyframes, `document.fonts`, `getBoundingClientRect`, `getAnimations()`, the network log. **The one rule that makes this work: every number you write into the clone was measured off the live page.** Screenshots are for human judgement and for giving each section agent a visual target — never for deriving a color, a gap, or a font size. Assets are mirrored byte-for-byte; text is copied verbatim.

Below, `cdp:` = `mcp__plugin_chrome-devtools-mcp_chrome-devtools__`. Every `evaluate_script`, `take_screenshot`, and `take_snapshot` passes `filePath`; not optional, it is what keeps a 400 KB style dump out of your context.

## Step 0 — precheck, flags, resume

Probe the server with `mcp__plugin_chrome-devtools-mcp_chrome-devtools__list_pages` (no params). It is the cheapest call that proves both "browser alive" and "full tool surface".

| Symptom | Action |
|---|---|
| `Could not find Google Chrome executable for channel 'stable'` | `ls /opt/google/chrome/chrome 2>/dev/null \|\| which google-chrome google-chrome-stable chromium`. If absent, give the user one line — `npx playwright install chrome` — and stop. Playwright MCP dies at the same missing binary; it is not a fallback for "no Chrome". |
| Only `navigate`/`evaluate`/`screenshot` exist | Server was started `--slim`. Stop and report; there is no snapshot, network, or emulation to degrade to. |
| ~9 read-only tools | Client is in read-only/plan mode. Exit plan mode, retry. |
| chrome-devtools *server* fails after those remedies | Announce a switch to `mcp__playwright__browser_*` and name the losses: no `emulate`, so no mobile/DPR/dark-mode pass, and no byte-exact response-to-disk. |

**If any browser call fails, or the page fights you: read `references/troubleshooting.md`** — precheck, failure→remedy table, lifecycle traps, and the Playwright fallback.

Resolve the target: a bare domain gets `https://`; a URL the user pasted mid-sentence still counts as the argument. Only ask a question if no URL is recoverable at all — everything else is inferred and reported, never negotiated. Then parse the argument string into `run.json.flags`. This is the complete flag set; do not invent more.

| Flag | Default | Effect |
|---|---|---|
| `--static` | off | Emit framework-agnostic `index.html` + `styles.css` + `main.js` + `sections/*.html`; tokens become `:root{--…}`. Overrides stack detection. |
| `--out DIR` | `./<domain>-clone` | Scaffold target. Ignored in adopt mode. |
| `--pages N` | `1` | Routes to clone. `--pages 0` = route-discovery report only, no build. |
| `--depth N` | `1` | Link-following hops during route discovery. |
| `--sections LIST` | all | Build only these ids (`03-features`, ranges `0-5`, or `role:header`). Others → `state:"skipped"` plus a placeholder comment in the page file. |
| `--max-parallel N` | `6` | Section agents per wave. Clamped to `[1,10]`; `1` = sequential in-thread. |
| `--viewport WxH` | `1440x900` and `390x844` | Repeatable. Explicit flags **replace** the default pair. First is the primary width, authoritative for the manifest; every listed viewport is measured and gated. |
| `--profile P` | `standard` | `cheap` \| `standard` \| `thorough`. |
| `--cheap` / `--thorough` | — | Aliases for `--profile cheap` / `--profile thorough`. |
| `--resume` | auto | Continue from `.clone/run.json`. Auto-implied when it exists and `phase != "done"`. |
| `--refresh` | off | Re-measure the live page before continuing a resumed run. |

`--profile` sets defaults an explicit flag then overrides (`--cheap --max-parallel 8` → 8). Dark-mode extraction is not a flag: it runs whenever `color.darkMode` shows signal.

| Dimension | `cheap` | `standard` (default) | `thorough` |
|---|---|---|---|
| Viewports measured | 1440 | 1440 + 390 | 1440 + 1024 + 390 |
| Section captures | desktop | desktop + mobile | + tablet |
| Segmentation `KEEP` | 3.6 (fewer, bigger) | 3.0 | 2.6 (finer cuts) |
| Hover / focus / active | skipped | CTAs + cards | every interactive element |
| Motion | static end-state only | keyframes + revealed sections | full `getAnimations()` + scroll sweep |
| Overlays / carousels | stubbed closed / slide 1 | measured if `open:true` / snap track | each opened / full behaviour |
| Asset mirror | images + fonts + **first-party js/css** | + svg, posters, sprites | + video, `srcset` variants, favicons, `og:` |
| Repair budget `K` | 1 | 3 | 4 |
| Full-page diff · `lighthouse_audit` | no · no | yes · no | per breakpoint · both sides |
| Typical cost | ~0.25× | 1× | ~2.5× |

This table is a **copy for immediate branching**; `references/scaling.md` owns it and carries the rows not listed here (`sectionAttemptsMax`, pages/depth, and why the 7-width fluid sweep is not a profile dial). Any change lands there first.

Resume: `run.json` lives in the **clone project root**, which in scaffold mode is `--out`, not your cwd — so look it up in this order and stop at the first hit: `./.clone/run.json`, `<--out>/.clone/run.json`, `./<domain>-clone/.clone/run.json`. On a hit, `cd` to that root and take every path from its `stack`; on more than one, use the newest `updatedAt` and say which you picked. Skipping this lookup silently starts a fresh run and re-measures the whole original — the one cost resume exists to avoid. Then: if `phase != "done"`, skip every `done` phase and **never re-measure the original** — measurement is the expensive half. Reset `running` sections to `pending`. Write `run.json` atomically (`.tmp` → `mv`) on every state transition, not just wave boundaries.

**Before you fan out, and any time the page is large: read `references/scaling.md`** — profiles, `filePath` discipline, `run.json` checkpointing, and `--resume`.

## Step 0b — is the target a site or an app?

Everything below measures **one page at rest**. If the target is an application — behind a login,
inside an iframe (Shopify App Bridge, embedded dashboards), or valuable for its **multi-step
flows** rather than its landing page — that shape is wrong and this pipeline quietly produces a
shallow answer. **Read `references/flows.md` instead**, and return here for the visual layer only
if the visual layer is genuinely the deliverable.

It covers operator-driven auth (a persistent profile, never credentials in a script), finding the
real document inside an iframe, discovering candidate flows, capturing a *state* rather than a
screenshot, and provoking the empty / validation / loading / error / resume states a happy-path
walkthrough misses. For an app the default deliverable is a **flow map**, not a stylesheet — §F0
says why, including when the target runs on a design system you should import rather than clone.

## Step 1 — navigate, and decide where the code goes

1. Motion instrumentation must exist before the page's own scripts do, so the HOOKS IIFE rides in on this navigation: **Step 2a — capture motion. Read `references/motion.md` §M0 now**: the `initScript` hooks must be installed before the page's own scripts run.
2. `cdp:new_page {url, isolatedContext:"clone"}` for a clean first-visit state, then `cdp:navigate_page {url, type:"url", initScript: HOOKS}`. `initScript` applies to the next navigation only — re-pass it after every reload. Navigation failures do not throw: read the response text for `Unable to navigate`.
3. `cdp:wait_for {text:["<a real string from the footer>"]}`, then poll `cdp:evaluate_script` for `document.readyState === 'complete'` and `document.fonts.status === 'loaded'` before measuring anything typographic.
4. Consent banner: `cdp:take_snapshot {filePath}`, `cdp:click {uid}` the accept control, re-snapshot. An overlay left up sets `body{overflow:hidden}` and destroys every `scrollHeight` you are about to read.
5. **Step 1b — decide the output stack. Read `references/stacks.md` now**: detect an existing repo's conventions or scaffold, then record the path map into `run.json.stack`. Adopt mode wins when a repo is detected; otherwise scaffold Next 15 App Router + Tailwind v4; `--static` overrides both.

Exit gate: `run.json` has `target.finalUrl`, `stack.mode` and the full path map, `phase:"prewarm"`.

## Step 2 — measure the original

**This order is the pipeline, not a suggestion.** The prewarm scroll, the fluid sweep, and `emulate` each destroy state an earlier pass reads: the prewarm fires every IntersectionObserver reveal and finishes every entrance animation, seven resizes re-run every media query, and `emulate` can trigger a reload that wipes `window.__cloneHooks` (installed only via `initScript`, on one navigation). So the at-load motion passes go **first**, cold, before anything scrolls.

| # | Pass | Instrument | Artifact |
|---|---|---|---|
| 1 | Motion at load (passes 1-4, 6) | install `scripts/extract-motion.js`, then `libs()` · `cssom()` · `transitions()` · `anims('load')` · `drain()` | `.clone/raw/motion-*.json` |
| 2 | Prewarm | install `scripts/extract-sections.js`, then `() => window.__clone.sections.prewarm()` | `.clone/raw/prewarm-1440.json` — check `imagesDecoded ≥ images·0.9` |
| 3 | Motion post-scroll (5, 7-9) | `scroll(21)` · `states()` · `states('<sel>',220)` · `page()` | `.clone/motion.json` |
| 4 | Foundation @ primary width | install `scripts/extract-foundation.js`, then `() => window.__clone.foundation.all()` | `.clone/raw/foundation-1440.json` → `.clone/foundation.json` |
| 5 | Fluid sweep | `cdp:resize_page` at 360/480/768/1024/1200/1440/1920, re-probe each. Seven widths at every profile — the solver needs ≥4 samples or the `clamp()` rails are invented | `.clone/raw/fluid-<w>.json` |
| 6 | Responsive (6 passes) | `scripts/probe-responsive.js`, ascending widths, scroll reset before each | `.clone/responsive.json` |
| 7 | Dark mode — last, it can reload | `cdp:emulate {colorScheme:"dark"}`, re-run, then restore `"auto"` | `.clone/raw/foundation-dark.json` |
| 8 | Network census | `cdp:list_network_requests {resourceTypes:[…], pageSize:300}` after the prewarm scroll | `.clone/assets.json` |

Skipping the prewarm costs a whole repair iteration and produces sections frozen at `opacity:0`. Draining the hooks after it produces motion data that describes the end state of every animation and the trigger of none. After **any** reload — including one `emulate` caused — re-`navigate_page {initScript: HOOKS}`, re-install all four payloads, re-prewarm, and record in `motion.json.warnings` anything the reload took before you drained it.

**Step 2a — capture motion. Read `references/motion.md` now** (you already read M0 before navigating): M1's pass table and the ordering rules that this table implements.

**Step 2a-ii — recover the motion system from source. Read `references/motion-source.md`** whenever `libs()` names GSAP/ScrollTrigger/Lenis/SplitType/Barba/Locomotive and no builder manifest was found. Runtime introspection cannot see a `once:true` trigger that already fired, a timeline for a section still below the fold, or a threshold that lives in an `if (x > N)` branch — their bundle can. This is also where the declarative motion vocabulary (`extract-motion-attrs.js`) is harvested so section agents can *declare* motion instead of inventing it.

**Step 2b — measure the foundation. Read `references/extraction.md` now**: it gives you the exact pass order, the `evaluate_script` invocations, and the merge that writes `.clone/foundation.json`.

**Step 2c — prewarm properly. `references/sectioning.md` §1 owns it** — the install sentence, the `imagesDecoded` check, and the quiescence poll. Read §1 now and the rest of that file at step 4a: prewarm now, segment later.

**Step 2d — sweep the breakpoints. Read `references/responsive.md` now**: the viewport plan, the probe set, and the `clamp()` solver that turns samples into authored CSS.

Exit gate: the three merged JSONs this step produces — `foundation.json`, `motion.json`, `responsive.json` — validate against their `schema` field (`sections.json` is written and gated at step 4a); `fonts.faces[]` is non-empty when the page uses webfonts; `patterns.breakpoints[]` is populated; `cssom.blocked[]` is either empty or backfilled from the network; and `fingerprint[]` exists (it is the numeric target verification diffs against).

## Step 3 — build the foundation

Mirror the bytes, then emit the tokens. Nothing downstream may reference a remote URL.

1. **Step 3b — mirror the assets. Read `references/assets.md` now**: the census, the byte-exact download path, and the `PROVENANCE.md` format.
2. **Step 3 — build the foundation. Read `references/foundation.md` now**: it maps every `foundation.json` field to the tokens, `@font-face`, and layout shell you must emit.
3. Emit yourself, or dispatch the single `clone-foundation` agent with absolute paths to `.clone/foundation.json` and `run.json.stack`. Either way the orchestrator owns the tokens file, the layout shell, the page file, `components/shared/**`, and `public/**` forever after.
4. Write `components/sections/_EXAMPLE.tsx` (or `sections/_EXAMPLE.html`). Every section agent reads it as the house style; a bad example multiplies by N.

Exit gate: `run.json.foundation.tokenCount > 0`; every family in `foundation.fontFamilies` has ≥1 local `woff2` and an `@font-face` whose `src` is a local path; zero requests to `fonts.googleapis.com` / `fonts.gstatic.com` / `use.typekit.net`; every mirrored file has a `PROVENANCE.md` row with URL, content-type, bytes, sha256.

## Step 4 — cut the page up and fan out

**Step 4a — cut the page into sections. Read `references/sectioning.md` now**: segmentation, per-section captures, dedup, and the wave plan.

**Step 4b — brief the team. Read `references/agent-brief.md` now**: the `spec.json` contract and the exact `PROMPT.md` each `clone-section` agent receives.

Segment at the primary width, score, retune `KEEP` if `sections.length` lands outside sane bounds, dedup by `contentHash`/`structureHash`, then write per section, under `.clone/sections/<NN>-<slug>/`: `spec.json`, `content.md`, `PROMPT.md`, and each gated width's capture in **both** formats back to back — `orig-w1440.webp` + `orig-w1440.png`, `orig-w390.webp` + `orig-w390.png`. The webp is the agent's visual target; the png is the only format `visual-diff.mjs` decodes, and shooting it later means re-navigating to the original at verify time.

Waves:

```
wave 0      orchestrator only — assets, tokens, shell, page file, _EXAMPLE
wave 1      shared chrome: SiteHeader, SiteFooter, hash-deduped shared components  (2-4 agents)
wave 2..n   body sections in document order, --max-parallel per wave (default 6)
wave n+1    orchestrator — drain requests.json, wire the page file, build
wave n+2    verify → repair waves (≤ K)
```

`references/sectioning.md` §10 owns the wave plan and the ownership map; the block above is the summary you dispatch from. Edit §10 first, then mirror it here.

Dispatch rules, all load-bearing:

- **One `Task` call per section, all of them in a SINGLE assistant message.** Sequential messages serialize the team and buy nothing.
- `subagent_type: clone-section`. Escalate that invocation to `model: opus` when the spec has `carousel != null`, `counts.forms > 0`, or `motion.scrollLinked.length > 0`.
- Each agent gets absolute paths only: `/abs/.clone/sections/<id>/PROMPT.md`, `spec.json`, `content.md`, both `.webp` captures, the tokens file, and `_EXAMPLE`. Never a URL.
- **One writer per file.** Each agent writes exactly one component file (two in `--static`: `sections/<id>.html` + `sections/<id>.css`) plus its own `report.md` and `requests.json`. A shared-chrome section's one file lives in `sharedDir` — one writer per *path*, never one directory per role. No agent gets an MCP tool, and `clone-section` agents get no `Bash` — the browser is a single serial resource and only you touch it.
- Above-the-fold sections (`box.y < 2·VH`) go in wave 2 regardless of order, so systemic errors surface early. Never put two sections with the same `contentHash` in one wave.
- Verify each wave while the next runs. A wrong container width caught at wave 2 is one edit to the tokens file instead of 20 agent repairs.
- At >24 sections warn once ("consider `--sections` for the top 12 first") and proceed. Do not gate.
- `--max-parallel 1`, or no `Task` availability: run the identical contracts in-thread, one section per turn. Same files, ~3× wall clock, `run.json` shape unchanged.

Multi-page (`--pages > 1`): routes were already discovered in phase `discover`, right after step 1's navigation and before the prewarm (it needs only a loaded DOM plus `curl`), and written to `.clone/pages.json`; `--pages 0` stops there — before any measurement — with the report. Measure each extra route into `.clone/pages/<pageId>/` (`pageId` = `home` for the entry route, otherwise the slugified path), then dedup across pages by `contentHash` before any wave is scheduled — promote a shared section to `components/shared/**` once and let each page file render it. Decisions land in `.clone/components.json`. Skipping this turns 5 pages into 5× the cost instead of ~2.2×.

## Step 5 — verify, repair, report

**Step 5 — verify and close the gap. Read `references/assembly.md` now**: the review order, every gate with its number, and the bounded repair loop.

Order is fixed and cheap-first: static sweep (grep) → drain `requests.json` → wire the page → build → capture the clone → diff → read only the failing sections' reports. Headline gates; the full table, every per-property tolerance, and the geometry probe matrix live in `references/assembly.md`.

| Gate | Pass | Fail |
|---|---|---|
| Section visual diff, desktop / mobile | ≤ 0.025 / ≤ 0.040 | > 0.060 / > 0.090 |
| Perceptual hash (8×8 aHash, Hamming/64) | ≤ 5 | > 10 |
| Worst tile (8×12 grid) | ≤ 0.20 | > 0.35 |
| Geometry probes | ≥ 90% pass, no probe \|Δy\| > 16px | < 85% |
| Text fidelity vs `content.md` | ≥ 0.995 | < 0.98 |
| Literal colors in section files | 0 | ≥ 1 |
| Arbitrary `-[Npx]` utilities | ≤ 3, each justified | > 8 |
| Console errors · HTTP ≥ 400 | 0 · 0 | ≥ 1 · ≥ 1 |
| Horizontal overflow at 360/390/768/1024/1280/1440/1920 | 0 offenders | ≥ 1 |
| Full-page diff mean · \|Δ docHeight\| | ≤ 0.030 · ≤ 4% | above |
| Build + typecheck — the commands in `run.json.stack` (per-stack table: `references/stacks.md` §7; `--static` has neither, so its gate is the console · 404 · overflow rows above against the served output) | exit 0 | anything else |

Repair loop, bounded: iterate only failing sections, `K` iterations max per the profile (1/3/4), `sectionAttemptsMax` per section. Fix systemic causes in the foundation yourself before dispatching any repair agent — a wrong base unit fails every section at once. Exit early on plateau (mean diff improves < 0.005 between iterations). Whatever is still failing goes into `.clone/UNRESOLVED.md` with its numbers; never claim a gate you did not measure. Then write `.clone/VERIFY.md` (worst-first) and `.clone/CLONE-REPORT.md` (gates, deviations, known limits), and set `phase:"done"`.

The mirror reproduces third-party text, marks, and imagery byte-for-byte; swapping or keeping them before you publish is your call — every mirrored file is listed in `.clone/PROVENANCE.md`.

## Hard rules

- Every `cdp:evaluate_script` / `take_screenshot` / `take_snapshot` passes `filePath`. The only exception is a small scalar you must branch on immediately.
- Never `Read` a dumped file whole. `Grep`/`jq` the field.
- Only the orchestrator touches Chrome. Subagents are pure file→file transducers.
- One writer per path. A crashed agent damages exactly one file.
- Text is copied verbatim from `content.md`. Improving the copy is a fidelity failure.
- Zero literal colors, font stacks, or magic px in section files — tokens only.
- `scripts/*.js` payloads are one arrow function, sent as `function`, installing onto `window.__clone`. Re-install after any `navigate_page`; `resize_page` does not wipe page state.
- `args` takes element uids only, never JSON. Interpolate literals into the function source.
- `emulate` resets every option you omit — restate the whole emulation state in every call.
- Measure with `scrollY === 0` and after `document.fonts.ready`.
- State limits instead of papering over them: `:active` is unverifiable, `prefers-reduced-motion` cannot be emulated, closed shadow roots are unreachable, rAF physics motion is fitted not recovered. Mark such output `authored-not-verified`.
- Screenshots never decide a number.

## Artifacts

Everything lives under `.clone/` in the clone project root.

| Path | Contents |
|---|---|
| `run.json` | Phase, flags, stack path map, per-section state, gates, budget. The only file `--resume` reads. |
| `foundation.json` · `sections.json` · `motion.json` · `responsive.json` | The four merged measurement truths for the entry page. |
| `assets.json` · `PROVENANCE.md` | Mirror manifest; every remote file with source URL, content-type, bytes, sha256. |
| `raw/` | Every `filePath` dump (`foundation-1440.json`, `vp-390.json`, `motion-cssom.json`, …). Scratch. |
| `css/` · `assets/` · `screenshots/` | Recovered cross-origin CSS; byte-exact mirror staging; page-level `orig-w<width>-full.{webp,png}` / `clone-w<width>-full.{webp,png}` plus `orig-w<width>-tile-<NN>.webp` for very tall pages. |
| `sections/<id>/` | `spec.json`, `content.md`, `PROMPT.md`, orig+clone captures (`.webp` for agents, `.png` for `visual-diff.mjs`), `report.md`, `requests.json`, `diff.json`, `geometry.md`. |
| `pages/<pageId>/` | `--pages > 1` only; same schemas per extra route. |
| `flows.json` · `flows/<flowId>/` · `FLOW-MAP.md` | App targets only (`references/flows.md`): per-flow state records, per-step captures, and the flow map. `profile/` holds a live session and is **always** gitignored. |
| `VERIFY.md` · `CLONE-REPORT.md` · `UNRESOLVED.md` | Gate table; final report; whatever the repair budget could not close. |

Close by linking `.clone/CLONE-REPORT.md` and `.clone/PROVENANCE.md`.

## Common failure modes

| Symptom | Cause | Go to |
|---|---|---|
| Sections measured at `opacity:0`, `translateY(40px)` | IntersectionObserver reveals never fired | `references/troubleshooting.md` — lifecycle traps |
| `Access denied: path … not within any of the configured workspace roots` | `filePath` outside the MCP roots and `os.tmpdir()` | `references/troubleshooting.md` — sandbox roots |
| Dumped file has the wrong extension (`.network-response`, `.json`) | the server substitutes its own extension | `references/troubleshooting.md` — forced extensions |
| `<binary data>` or `<Response body not available anymore>` | inline body read, or Chrome evicted it | `references/assets.md` — `responseFilePath` and the `curl` path |
| `Element not found` for a uid | DOM mutated since the snapshot | `references/troubleshooting.md` — re-snapshot before uid calls |
| 30 candidate sections on one page | `KEEP` too low | `references/sectioning.md` — retune and dedup |
| Every section fails the same geometry probe | foundation is wrong, not the sections | `references/assembly.md` — systemic-cause first |
| Mobile or dark values missing entirely | running on the Playwright fallback (no `emulate`) | `references/troubleshooting.md` — fallback map |
