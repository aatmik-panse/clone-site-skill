# Sectioning — cutting the page into clone-able units

Shorthand in this file: `cdp:` = `mcp__plugin_chrome-devtools-mcp_chrome-devtools__`.

Output: `.clone/sections.json` (`clone-site/sections@1`) for the entry page, `.clone/pages/<pageId>/sections.json`
per extra page, two captures per section per width, and the wave plan in `run.json.sections[]`. The per-section
agent contract is built from these in `references/agent-brief.md`.

## 1. Prewarm — never segment a cold page

Lazy sections, IntersectionObserver reveals and `loading=lazy` images make a cold DOM under-report document
height by 30–60%. Prewarm first, always, at the primary width.

Read `scripts/extract-sections.js` with `Read` and pass its **entire contents** as the `function` argument to
`mcp__plugin_chrome-devtools-mcp_chrome-devtools__evaluate_script`, with `filePath` set to the run-directory
target for that pass (`.clone/raw/install-sections.json`). Never inline the payload into this reference; never
call it without `filePath`. Re-install after **any** `navigate_page` (including `reload`); `resize_page` and
`emulate` keep page state.

| Step | Call | Check |
|---|---|---|
| install | the sentence above | `{ok:true, installed:[…]}` |
| prewarm | `cdp:evaluate_script {function:"() => window.__clone.sections.prewarm()", filePath:".clone/raw/prewarm-1440.json"}` | `imagesDecoded ≥ images·0.9` |
| quiescence | `cdp:list_network_requests {resourceTypes:["image","font","stylesheet"], pageSize:200}` | nothing pending |
| segment | `cdp:evaluate_script {function:"() => window.__clone.sections.all()", filePath:".clone/raw/sections-1440.json"}` | the manifest |

`prewarm()` is `async`; `evaluate_script` awaits it. `docHeight` under 1.5 viewports on a marketing page means
reveals are still gating content — fix that via `references/troubleshooting.md` before trusting a single box.

## 2. The boundary score

`all()` walks the content root's children, scores each candidate on six independent signals, descends into
anything scoring under `KEEP`, then merges slivers into the previous keeper.

| Signal | Weight | Why it is evidence |
|---|---|---|
| full-bleed width (`w ≥ 0.98·VW`) | **2.0** (1.0 at ≥ 0.70·VW) | sections span the page; inner content does not |
| vertical extent `h/VH` | **1.5** ≥0.9 · **1.0** ≥0.3 · **0.4** ≥0.12 | a section is a screenful of scroll |
| background/border/text-color change vs previous sibling | **1.5** | the strongest human cue for "new section" |
| semantic tag (`section header footer main article aside nav form`) | **1.5** | free ground truth when the author bothered |
| heading inside (`h1..h3`, `[role=heading]`) | **1.0** | sections announce themselves |
| name hint in `id`/`class`/`data-section`/`aria-label` | **0.75** | `class="hero"` is not a coincidence |
| ≥2 element children | **0.25** | tiebreak against wrappers around a single node |

- **Descend** (depth < 3) when `score < KEEP` **and** ≥2 children **and** `h > 0.6·VH`: a low score on a tall node
  means the sections are one level deeper, not that there are none. **Split** a keeper with `h > 3.5·VH` holding
  ≥2 `h1..h3` the same way — with no coverage test, which is trap (c).
- **Keep** everything else whatever its score: a node that cannot be descended into is emitted. `KEEP` gates
  descent and sliver-merging, not admission.
- **Merge** a sub-`KEEP` node with `h < SLIVER·VH` (default 0.15) and no heading into the previous section as an
  `extraSelectors[]` entry — dividers, logo strips, spacer bands. Never into chrome (`header`/`footer`/`nav`/fixed),
  never a fixed/sticky node itself. Anything containing a heading is never a sliver, which is why duplicate overlay
  headings survive (trap (d)).
- Candidates are ordered by **document position** — `getBoundingClientRect().top` ascending, not DOM order — so an
  absolutely-positioned block lands where the reader sees it.
- Scroll root resolves first: `<html>`, else the largest genuinely-scrolling `overflow-y:auto|scroll` container
  (Lenis / Locomotive / ScrollSmoother hijack). Every `box` is **scroll-root-relative**; `scrollRoot` records which.
  Content root then unwraps up to 6 single-child shells (`#__next > div > div`) so scoring starts at the real
  section list — trap (a) is when that is not enough.

## 3. Retune, and what `KEEP` actually is

The knob SKILL.md calls `KEEP` is the payload's `keep` option — `sections.all({keep, sliver})`, echoed back as
`sections.json.keepThreshold`. Default `3.0`. Per-profile values (`cheap` 3.6 · `standard` 3.0 · `thorough` 2.6)
live in `references/scaling.md`.

Be honest about direction: `keep` is the score a node must reach to be **accepted instead of descended into**, so
raising it does two opposite things — more descent into low-scoring tall blocks (**more**, smaller sections) and
more sliver merging (**fewer**, bigger sections). On a page with semantic tags and headings almost nothing scores
under 3.6, so only the merge moves and "3.6 → fewer, bigger" holds; on div soup, raising `keep` cuts deeper. Never
predict the count — read `stats.kept` / `stats.retunes` from the dump after each pass. `sliver` is the
one-directional dial: raise it to 0.22 for reliably fewer, bigger sections.

`all()` auto-retunes up to twice, **only when `keep` is omitted**:

| Observed | Payload does | Read it as |
|---|---|---|
| `kept < 4` | `keep` → 2.4, then 1.8 | prewarm failed, or one giant node (traps a–c) — check those before accepting |
| `kept > 30` | `keep` → 3.6, then 4.2, `sliver` → 0.22 | it is cutting inside sections |
| `4 ≤ kept ≤ 30` | accept | a marketing page is 6–24 |

`cheap` and `thorough` pass `keep` explicitly and therefore get **no** auto-retune; you own the loop:

```
cdp:evaluate_script {function:"() => window.__clone.sections.all({keep:3.6,sliver:0.22})",
                     filePath:".clone/raw/sections-1440.json"}
```

`args` carries element uids only, so option literals are interpolated into the `function` source; there is no JSON
channel. Re-cut instead of re-tuning when a `box.h` exceeds 40% of `docHeight`, when two adjacent sections share a
`contentHash`, or when a `box.w` is under half the viewport (inner content that scored on background change alone).

## 4. Four traps, each of which silently yields one giant section or drops content

Run this audit once, right after the first `all()`; it reads the element cache `all()` leaves on
`window.__clone.state.sectionEls`, so same page state, no re-walk:

```
cdp:evaluate_script {filePath:".clone/raw/section-audit-1440.json", function:
"() => { const U = window.__clone.util, E = window.__clone.state.sectionEls || [], R = Math.round, VH = innerHeight;
  const b = (e) => e.getBoundingClientRect();
  const contents = [...document.querySelectorAll('body *')].filter((e) => getComputedStyle(e).display === 'contents')
    .slice(0, 40).map((e) => ({ sel: U.cssPath(e), tag: e.tagName, kids: e.children.length }));
  const audit = E.map((s) => { const r = b(s.el), k = [...s.el.children].filter((c) => b(c).height > 8);
    const cover = r.height ? k.reduce((n, c) => n + b(c).height, 0) / r.height : 0;
    const st = [...s.el.querySelectorAll('*')].filter((c) => getComputedStyle(c).position === 'sticky');
    return { id: s.id, h: R(r.height), vh: +(r.height / VH).toFixed(2), coverage: +cover.toFixed(2), kids: k.length,
      sticky: st.length, stickyTallest: st.length ? R(Math.max(...st.map((x) => b(x).height))) : 0,
      pinned: r.height > VH * 2 && cover < 0.8 && st.length > 0 }; });
  const dupes = [];
  for (const a of E) for (const c of E) { if (a === c) continue; const ra = b(a.el), rc = b(c.el);
    if (!rc.width || !rc.height || ra.width * ra.height <= rc.width * rc.height) continue;
    const ox = Math.max(0, Math.min(ra.right, rc.right) - Math.max(ra.left, rc.left));
    const oy = Math.max(0, Math.min(ra.bottom, rc.bottom) - Math.max(ra.top, rc.top));
    if (ox * oy < rc.width * rc.height * 0.9) continue; const t = U.nlow(c.el.innerText);
    dupes.push({ inner: c.id, outer: a.id, domContained: a.el.contains(c.el),
      textSubset: !!t && U.nlow(a.el.innerText).indexOf(t) >= 0 }); }
  return { docHeight: R(document.documentElement.scrollHeight), contents: contents, audit: audit, dupes: dupes,
    sumSectionH: E.reduce((n, s) => n + R(b(s.el).height), 0) }; }"}
```

**(a) No `<main>` and no `<section>` anywhere.** Framer, Webflow and hand-rolled Next output nest everything in
`<div>`s and render headings as `<div class="framer-text">`, so the semantic-tag (1.5) and heading (1.0) signals go
to zero, the monster-split test (≥2 `h1..h3`) never fires, and the page returns as one candidate. Symptoms:
`kept ≤ 2`, one section with `h ≥ 0.6·docHeight`, or `contentRoot` still `body`. Do **not** walk `body.children`
yourself. The real content root is the deepest node with `scrollHeight ≥ 0.9·docHeight` and ≥3 visible children;
compare it to the manifest's `contentRoot`, then force descent by raising `keep` just past the giant's own recorded
`score` (`{keep: <score> + 0.1}`) — one level per pass, depth capped at 3 below the content root, re-reading
`stats.kept` each pass. If the giant has <2 children the payload cannot descend at all (a single-child chain deeper
than the 6-shell unwrap): flatten those wrappers with the `replaceWith` call below, then re-run `all()`.

**(b) `display: contents` wrappers.** Layout-transparent by design, so `getBoundingClientRect()` is `0×0` while
children lay out normally. The payload's floor (`h < 24 || w < 40`) skips such a node **and never descends into
it**, so its whole subtree disappears from the manifest. Symptom: `contents[]` non-empty with
`sumSectionH < 0.8 × docHeight`, or `kept` 0–1 when the wrapper sits high. Fix by unwrapping — replacing a
`display:contents` element with its children is layout-neutral, that is what the value means:

```
cdp:evaluate_script {filePath:".clone/raw/unwrap-contents.json", function:
"() => { const h0 = document.documentElement.scrollHeight, out = [];
  for (const e of [...document.querySelectorAll('body *')].filter((x) => getComputedStyle(x).display === 'contents'))
    { out.push(window.__clone.util.cssPath(e)); e.replaceWith(...e.childNodes); }
  return { unwrapped: out, before: h0, after: document.documentElement.scrollHeight }; }"}
```

Guards: run it after Step 2's measurement passes and before `all()` — it mutates the DOM, so selectors recorded
earlier go stale and `--refresh`/`--resume` must redo it. If `after` differs from `before` by >1%, the page's CSS was
matching those wrappers as children: `cdp:navigate_page {type:"reload"}`, re-install, accept the coarser cut, and
add a `warnings[]` row naming the selectors.

**(c) A scroll-pinned section** — a very tall track (5–10 viewports) wrapping a short `position:sticky` stage, where
scroll drives the stage's contents. The monster-split fires on height plus two headings with **no coverage test**,
so it descends and returns the sticky stage plus a pile of spacer divs: the effect is destroyed and most of the page
height vanishes. Only split a tall block when its children's summed height actually covers it — `coverage ≥ 0.8`.

| Audit row | Verdict | Action |
|---|---|---|
| `vh ≥ 3.5`, `coverage ≥ 0.8` | genuine tall stack | the split is correct |
| `vh ≥ 2`, `coverage < 0.8`, `sticky ≥ 1` (`pinned:true`) | scroll-pinned track | keep whole. If `all()` split it, restore the parent as the section, move fragment selectors into `extraSelectors[]`, re-probe with `sections.probe()`, add `warnings:["pinned:<id> re-merged"]` |
| `vh ≥ 2`, `coverage < 0.8`, `sticky` = 0 | absolutely-positioned layout | keep whole; take boxes from the audit, never from summing children |

A pinned section is a one-agent, `model: opus` job whose brief carries the track height, `stickyTallest`, and the
`scrollLinked` slice from `motion.json` (rebuild recipes: `references/motion.md`). Never let the track height come
from a screenshot.

**(d) A duplicate heading positioned absolutely over a section as a SIBLING.** Page builders do this constantly.
`Node.contains()` misses it, it holds a heading so it is never sliver-merged, and you ship the H1 twice. Use
geometry: a `dupes[]` entry with ≥90% of the inner box inside the outer box and `textSubset:true` is a duplicate —
drop `inner`, append its selector to the outer section's `extraSelectors[]`; `domContained:true` rows are the same
thing one level down, same treatment. Never merge two overlapping candidates whose text differs: that is a real
overlay and it keeps its own entry.

## 5. Roles, chrome, overlays

`role` is a closed vocabulary: `header` `footer` `nav` `section` `sticky-cta` `overlay`.

| Detected as | Rule in the payload | What the pipeline does |
|---|---|---|
| `header` | `<header>`, or fixed/sticky with `top ≤ 8` + ≥2 links + `h < 0.3·VH`, or first block with ≥2 links + `h < 0.25·VH` | `shared:true`, built once, mounted by the layout not the page. `sticky` carries the exact `position` so offset behaviour survives. |
| `footer` | `<footer>`, or last block with ≥4 links | `shared:true`, layout-mounted |
| `nav` | `<nav>` outside header/footer | own section unless its `contentHash` matches the header's |
| `sticky-cta` | `position:fixed`, `z-index ≥ 40`, `h < 0.5·VH`, not at the top | `shared:true`, layout-mounted, excluded from body flow so it cannot double-render |
| `overlay` | `[role=dialog]`, `[aria-modal=true]`, `<dialog>` | never a body section; listed in `overlays[]` including `display:none` ones, with `open`, `zIndex`, guessed `triggerSelector` |
| `section` | everything else | one `clone-section` agent, one file |

Overlay policy: on `cheap`/`standard` only `open:true` overlays get a section, the rest ship closed with the measured
decl. On `--thorough` open each for a second pass — `cdp:take_snapshot {filePath:".clone/raw/snapshot-overlays.txt"}`
for the trigger uid, `cdp:click {uid}`, re-run `all()`, `cdp:press_key {key:"Escape"}`. A cookie banner is an overlay
to dismiss before **any** capture, never a section.

## 6. Repeated grids, carousels, forms

| Field | Detection | What the brief says (filled in `references/agent-brief.md`) |
|---|---|---|
| `repeat` | a parent with ≥3 children where ≥80% share `tag + first 3 classes + child count`; carries `count`, `containerSelector`, `itemSelector`, `columns` (distinct rounded `left`s), `gap`, `display`, `itemHash` | one local sub-component + a data array. 12 cards = 1 card + 12 data objects, never 12 literals. |
| `carousel` | library class/attr (`swiper slick embla keen-slider flickity glide splide owl-carousel tns- marquee`), `aria-roledescription="carousel"`, or `scrollWidth > clientWidth+24` with `overflow-x` set and ≥3 children | rebuild as a CSS `scroll-snap` track first. A JS carousel library is a `requests.json` entry, never an agent-side install. |
| `form` | `counts.forms > 0` (the payload counts `form,input,textarea,select` per section). Fill it from the DOM: per field `type`, `name`, `required`, `autocomplete`, `inputmode`, verbatim `placeholder`, label text and whether the label is visible, the resting box, and the computed `appearance`; plus every `interactive[].state` row from `motion.json` whose selector is inside the form | §5a — field attributes are content, `appearance:none` is preserved with its affordance re-authored, `::placeholder` styled explicitly, and every captured state ships including the `authored-not-verified` ones. Escalates the section to `model: opus`. |
| `repeat-item` probe | first item of the winning group | per-card geometry that verification re-measures |

A section with `repeat.count ≥ 3` also gets `content()` rows prefixed `item[<i>].` — that prefix is what keeps card
3's body on card 3.

## 7. Per-section captures

Never crop a full-page PNG: you lose resolution and gain off-by-one bands. Resize the viewport to the section and
shoot it, per section, at the current width:

```
cdp:evaluate_script {function:"() => window.__clone.sections.capture(\"#features\",{hideFixed:true})",
                     filePath:".clone/raw/capture-03-features.json"}
cdp:resize_page     {width:1440, height:<min(needsHeight, 4000)>}
cdp:evaluate_script {function:"() => window.__clone.sections.capture(\"#features\",{hideFixed:true})",
                     filePath:".clone/raw/capture-03-features.json"}      # layout moved; re-align
cdp:take_screenshot {filePath:".clone/sections/03-features/orig-w1440.webp", format:"webp", quality:88}
cdp:take_screenshot {filePath:".clone/sections/03-features/orig-w1440.png",  format:"png"}
cdp:evaluate_script {function:"() => window.__clone.sections.restore()", filePath:".clone/raw/restore.json"}
```

**Both formats, back to back, before `restore()`.** `.webp` at quality 88 is what the section agent and any repair
agent read; `.png` is the only thing `visual-diff.mjs` can decode (node built-ins, no WebP decoder). Shooting the
PNG later is not an option — it would mean re-navigating to the original and re-prewarming it during the verify
phase, when the browser is pointed at `localhost`, which is exactly the "never re-measure the original" rule this
pipeline is built on. Two `take_screenshot` calls from one identical viewport/scroll state, every section, every
gated width. The clone side does the same in `references/assembly.md` §4.

**Page level, once per gated width, before the per-section loop** (skip under `--profile cheap`, where the
full-page diff gate does not run). Same both-formats rule, and the same reason it cannot wait:

```
cdp:take_screenshot {fullPage:true, format:"webp", quality:88, filePath:".clone/screenshots/orig-w1440-full.webp"}
cdp:take_screenshot {fullPage:true, format:"png",              filePath:".clone/screenshots/orig-w1440-full.png"}
```

`fullPage` is incompatible with `uid`. Pages past roughly 16000 px come back clipped — that is what the stitched
`.clone/screenshots/orig-w<w>-tile-<NN>.webp` set in `references/scaling.md` exists for; record the fallback in
`run.json.notes`. `references/assembly.md` §5.3 diffs `orig-w<w>-full.png` against `clone-w<w>-full.png`.

`capture()` hides every other fixed/sticky box, pauses animations, forces instant scrolling, aligns the section top
to the viewport top, and returns `{box, needsHeight, clipped, tall, atTop, hiddenCount}`. Pass `{hideFixed:false}`
when the section **is** the chrome, `{offset:<headerHeight>}` when you want the sticky header in frame. Always
`restore()` before the next section — hidden chrome and the paused-animation stylesheet are global, and skipping it
smears a band across every later shot. `atTop:false` on the second `capture()` means the container fought you
(hijack, snap, focus trap): shoot anyway, record the delta, read the scroll-hijack row in
`references/troubleshooting.md`. Tall sections (`tall:true`, `needsHeight > 4000`), including every pinned track from
trap (c): resize to `1440×3600` and shoot bands via `capture("#sel",{offset:-3600*k})` for `k = 0,1,2` into
`.clone/sections/<id>/orig-w1440-tile-<NN>.webp` **and `-tile-<NN>.png`** (same pairing, same reason), capped at 3
tiles plus a `warnings[]` row.

Cheaper route where it applies: `cdp:take_snapshot {filePath:".clone/raw/snapshot-<label>.txt"}`, grep the landmark's
uid, then `cdp:take_screenshot {uid, filePath:…, format:"webp", quality:88}` followed by the `format:"png"` sibling
from the same uid. Real limits — only a11y-tree nodes have
uids (a bare `<div>` has none), `uid` is incompatible with `fullPage`, sticky chrome is still painted, and a very
tall element comes back clipped. Use it for `header`/`footer`/`nav`, confirm the file is >5 KB, else fall back.

Width order: capture every section at the primary width, then switch viewport **once** — `cdp:emulate
{viewport:"390x844x3,mobile,touch"}` → re-run `all()` (mobile reorders sections, it does not remove them) → capture
each to `orig-w390.webp` + `orig-w390.png` → restore desktop explicitly with `cdp:emulate {viewport:"1440x900x1"}` plus
`cdp:resize_page {width:1440,height:900}`. Never interleave widths: viewport changes are the expensive call.

## 8. Merging widths into `sections.json`

`all()` returns one width's truth, keyed by integer width as a string. The orchestrator merges:

1. The **primary width** manifest is the skeleton — it owns `id`, `order`, `role`, `label`, `selector`, `probes`,
   `contentHash`, `structureHash`.
2. For each other width, match by `contentHash`, else `structureHash`, else `order`; merge only the width-keyed maps
   (`box`, `background`, `layout`, `docHeight`) and set `widths`. Unmatched secondary sections become
   `warnings:["w390: unmatched <selector>"]` — a breakpoint-only element, owned by `references/responsive.md`.
3. Fill what the page cannot know: `page.id`, `shared`/`sharedKey`/`dedupOf` after dedup (§9), `keepThreshold` from
   the accepted pass, and the §4 trap corrections.
4. Write `.clone/sections.json`; extra pages to `.clone/pages/<pageId>/sections.json`, identical schema. Never `Read`
   the merged file whole afterwards — `Grep`/`jq` the field.

Ids are stable for the run's life: `<NN>-<slug>`, zero-padded document order at the primary width, deduped with a
numeric suffix. Never renumber after `run.json` is written, including on `--resume`.

## 9. Hashing, cross-page dedup, promotion

`contentHash` = tag skeleton + normalized text + background key. `structureHash` = skeleton + background + child
count + height bucket. Identical text ⇒ same `contentHash`; same layout with different copy ⇒ same `structureHash`
only. For fuzzy similarity dump 4-gram shingles of the tag skeleton — `cdp:evaluate_script {function:"() =>
window.__clone.sections.skeletons()", filePath:".clone/raw/skeletons-1440.json"}` (needs `all()` first, same page
state) — and compute Jaccard in Node from the dumps, never in-browser: it is an O(n²) cross-page comparison.

Multi-page runs (`--pages > 1`) discover routes cheapest-source-first and union the results: (1) `robots.txt`
`Sitemap:` lines via `Bash curl -s <origin>/robots.txt`, regex `^Sitemap:\s*(\S+)`; (2) `sitemap.xml` /
`sitemap_index.xml`, parsing `<loc>` and following index children (cap 5); (3) nav + footer links via the payload
below; (4) every same-origin `<a href>`, same payload, `--depth` hops; (5) `__NEXT_DATA__.buildManifest` or
`/_next/static/*/_buildManifest.js`, which reveals routes with no inbound link.

```
cdp:evaluate_script {filePath:".clone/raw/routes.json", function:
"() => { const org = location.origin, seen = new Map();
  const clean = (h) => { try { const u = new URL(h, location.href); if (u.origin !== org) return null;
    u.hash = ''; u.search = ''; return u.pathname.replace(/\\/+$/, '') || '/'; } catch (e) { return null; } };
  for (const a of document.querySelectorAll('a[href]')) { const p = clean(a.getAttribute('href')); if (!p) continue;
    const z = a.closest('header,nav') ? 'nav' : a.closest('footer') ? 'footer' : 'body';
    const r = seen.get(p) || { route: p, count: 0, zones: [], text: (a.innerText || '').trim().slice(0, 60) };
    r.count++; if (r.zones.indexOf(z) < 0) r.zones.push(z); seen.set(p, r); }
  return { routes: [...seen.values()] }; }"}
```

Drop `mailto:`/`tel:`/`javascript:`, non-HTML extensions, infinite spaces (`/tag/`, `/page/\d+`), and anything
`robots.txt` disallows. Rank `nav zone > footer zone > inbound count > sitemap order`, take the top `--pages N`, write
`.clone/pages.json` = `{origin, discovered, selected, depth, pages:[{id, route, url, rank, reason, state,
sectionCount, manifest}]}`. Segment **every** selected page before building **any** of them: building `home` first
and then finding that `pricing` shares four of its sections means rewriting `home`'s imports.

Promotion into the shared dir, recorded in `.clone/components.json` — same shape discipline as `pages.json`, keyed
`c_<hash>` where `<hash>` is the `contentHash` (or `structureHash` for a props-derived promotion) that grouped the
members:

```jsonc
{ "schema": "clone-site/components@1", "capturedAt": "s",
  "components": { "c_9f2c41a8": {
    "kind": "content|structure|jaccard|role",   // which row of the table below promoted it
    "sharedKey": "SiteHeader",                  // matches run.json.sections[].sharedKey
    "file": "components/shared/SiteHeader.tsx", // repo-relative, orchestrator-owned
    "pages": ["home", "pricing"],               // pageIds it appears on
    "memberSections": ["00-header", "pricing:00-header"],
    "props": [{ "name": "ctaLabel", "from": "differing text node .cta > span", "values": { "home": "Start free", "pricing": "Get started" } }],
    "similarity": 0.94                          // Jaccard, only for kind:"jaccard"
  } } }
```

| Condition | Result |
|---|---|
| identical `contentHash` on ≥2 pages | shared, no props, built once |
| `role ∈ {header, footer, sticky-cta}` anywhere | shared + layout-mounted, even at 1 occurrence |
| identical `structureHash`, differing `contentHash`, ≥2 pages | shared + props derived from the differing text nodes |
| Jaccard ≥ 0.80 across ≥3 pages | shared + props |
| same `structureHash` ≥3× **within one page** | page-local archetype component + a data array, not shared |

Everything else stays page-local. 5 pages × 8 sections typically collapses from 40 candidates to ~22 builds; that
collapse is the only reason multi-page is affordable.

## 10. How many sections, and the wave plan

Sanity bounds 6–24. Above **24** warn once — "24+ sections; consider `--sections 0-11` for the top half first" — then
proceed. Do not gate. Budgets and profiles: `references/scaling.md`.

| Wave | Who | Work |
|---|---|---|
| 0 | orchestrator only | tokens, base CSS, `@font-face`, layout shell, `_EXAMPLE`, asset mirror, empty page file |
| 1 | 2–4 agents | shared chrome: header, footer, sticky-cta, hash-deduped shared components |
| 2…n | `--max-parallel` agents (default 6, clamp `[1,10]`) | body sections in document order |
| n+1 | orchestrator | drain `requests.json`, wire the page file, build |
| n+2… | orchestrator + repair agents | verify per `references/assembly.md` |

Every above-the-fold section (`box.<primary>.y < 2·VH`) goes in the first body wave, so a systemic error surfaces at
wave 2 instead of wave 6. Never schedule two sections with the same `contentHash` or `structureHash` in one wave —
dedup first, then schedule. Verify each wave while the next runs: a wrong container width caught at wave 2 is one
orchestrator edit, at wave 6 it is 20 agent repairs. Write `run.json` on every section state transition.

Ownership at fan-out — one writer per path, so no locking and no merge conflicts:

| Path | Owner |
|---|---|
| tokens file (`app/globals.css`, or `styles.css` in `--static`), base CSS, layout shell, page file (`index.html` in `--static`, including its per-section `<link>` tags), config, lockfile | orchestrator |
| asset dir, `PROVENANCE.md`, `.clone/**` except the two agent files | orchestrator |
| shared components dir — **except** one file per wave-1 agent's own `spec.writeFile` | orchestrator |
| one section component file — `components/sections/<Name>.tsx`, **or `components/shared/<Name>.tsx` when the section is shared chrome** (wave 1: `role: header\|footer\|sticky-cta`, or a hash-deduped shared component — those specs' `writeFile` points into `sharedDir` by construction), or in `--static` **both** `sections/<id>.html` and `sections/<id>.css` (every rule scoped to `[data-section="<id>"]`) — plus `.clone/sections/<id>/report.md` and `requests.json` | exactly one `clone-section` agent |

The agent-facing wording of this map is in `references/agent-brief.md`; the static-mode mechanics are in
`references/stacks.md` §5.

## 11. Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| every `box.y` = 0 | measured inside a hijacked scroll container | boxes are container-relative by design; confirm `scrollRoot` before "fixing" it |
| section shots all show the same band | `restore()` not called between sections | one `capture()` → one screenshot → one `restore()` |
| mobile manifest has different ids | you adopted the secondary width's ids | secondary widths never own ids; merge by hash into the primary skeleton |
| `docHeight` grows every pass | infinite scroll, or a marquee appending nodes | segment from one prewarmed pass, note it in `warnings[]` |
| `sumSectionH` ≪ `docHeight` | trap (b), or a section dropped by the sliver merge | run the §4 audit; check `extraSelectors[]` before re-cutting |
