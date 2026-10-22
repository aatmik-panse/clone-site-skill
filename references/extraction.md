# Extraction — reading the foundation off the live page

Shorthand in this file: `cdp:` = `mcp__plugin_chrome-devtools-mcp_chrome-devtools__`.

Output of this phase: `.clone/foundation.json` (`clone-site/foundation@1`), merged from
`.clone/raw/foundation-<primaryWidth>.json`, seven `.clone/raw/fluid-<w>.json`, `.clone/raw/foundation-dark.json`,
and any stylesheet text recovered into `.clone/css/`. Token emission is `references/foundation.md`; this file only
produces numbers. Prewarm before you measure anything (`references/sectioning.md`) — a cold page under-reports
document height by 30–60% and freezes revealed sections at `opacity:0`.

## 1. The `evaluate_script` contract

| Rule | Detail |
|---|---|
| One function, as a string | The server evaluates `` `(${function})` `` and calls it. `() => …`, `async () => …`, `function(){…}` work; a bare expression (`document.title`) does not. |
| Installer, not payload | `scripts/extract-foundation.js` is **one arrow function** that attaches `window.__clone.foundation` + `window.__clone.util` and returns `{ok:true,installed:['util','foundation'],v:1}` (~60 bytes). Every later pass is a one-line invocation. |
| Re-install after navigation | `window.__clone` dies on any `cdp:navigate_page`, including `type:"reload"`. `cdp:resize_page` and `cdp:emulate` do **not** wipe page state — but `emulate` can trigger a reload, so re-check `window.__clone.v` before trusting a measure. |
| `args` is element uids only | Each entry is resolved to a live `ElementHandle`; `"1200"` or `'{"a":1}'` throws *element not found*. Interpolate literals into the function source instead, or park them on `window.__clone.state` in a prior call. |
| Always `filePath`, parse defensively | Returns are `JSON.stringify`'d in-page with no size cap; `foundation.all()` is 150–600 KB. `filePath` forces a `.json` extension and must sit inside the MCP workspace roots (or `os.tmpdir()`). Strip any wrapper with `JSON.parse(text.slice(text.indexOf('{'), text.lastIndexOf('}') + 1))`, and `Grep`/`jq` the field — never `Read` the dump whole. |

Never return a DOM node, `NodeList`, `Map`, `Set`, `CSSStyleDeclaration`, `FontFaceSet`, `CSSRule`, `Animation`, or
anything circular — you get `{}`, an index→property-name map, or a `TypeError`; `undefined` serializes to the string
`"undefined"` and `NaN`/`Infinity` to `null`. The extractor returns plain objects of primitives, caps every collection
(`caps` is echoed in the output), and appends to `warnings[]` instead of throwing.

## 2. Pass order

Read `scripts/extract-foundation.js` with `Read` and pass its **entire contents** as the `function` argument to
`mcp__plugin_chrome-devtools-mcp_chrome-devtools__evaluate_script`, with `filePath` set to the run-directory target for
that pass (`.clone/raw/install-foundation.json`). Never inline the payload into this reference; never call it without
`filePath`.

| # | Pass | Call | Artifact |
|---|---|---|---|
| 1 | install | the sentence above | `.clone/raw/install-foundation.json` |
| 2 | foundation @ primary width | `{function:"() => window.__clone.foundation.all()", filePath:".clone/raw/foundation-1440.json"}` | the whole schema, `type.fluid` and `darkMode.tokenDiff` still empty |
| 3 | network census | `cdp:list_network_requests` — `references/assets.md` | `.clone/assets.json` |
| 4 | blocked-sheet backfill | §3, one `cdp:get_network_request` per `cssom.blocked[]` | `.clone/css/<slug>.css` |
| 5 | fluid sweep, 7 widths | §7 | `.clone/raw/fluid-<w>.json` |
| 6 | solve | one `fitFluid` call, §7 | `.clone/raw/fluid-solved.json` |
| 7 | dark | §5 | `.clone/raw/foundation-dark.json` |
| 8 | merge | §11 | `.clone/foundation.json` |

Pass 2 verbatim, plus the two cheap re-reads the same install provides:

```jsonc
{ "function": "() => window.__clone.foundation.all()",         "filePath": ".clone/raw/foundation-1440.json" }
{ "function": "() => window.__clone.foundation.vars()",        "filePath": ".clone/raw/vars-dark.json" }
{ "function": "() => window.__clone.foundation.fingerprint()", "filePath": ".clone/raw/fingerprint-clone-1440.json" }
```

`all()` is `async` (it awaits `document.fonts.ready`, capped at 1500 ms) and `evaluate_script` awaits the promise;
`vars()` only sees non-`:root` scopes after `all()` has built the CSSOM declaration index on this page. Before pass 2,
confirm `document.readyState === 'complete'`, `document.fonts.status === 'loaded'`, `scrollY === 0`.
**`document.fonts.ready` must settle before any typographic read**: until it does every text node is laid out with the
fallback face, so `fontSizePx` survives but `lineHeightPx` (when `normal`), `textWidthPx` and `resolvedFamily` are all
measured off the wrong font.

## 3. CSSOM walking

The walker (`walkRules`/`readSheet`) covers `document.styleSheets`, `document.adoptedStyleSheets`, every open
`shadowRoot`'s `adoptedStyleSheets` and inline `<style>`, and recurses through `@import`. Five hazards, all handled —
know them so you can read the warnings:

| Hazard | Symptom | Handling |
|---|---|---|
| Cross-origin sheet | `.cssRules` throws `SecurityError` | listed in `cssom.sheets[]` with `readable:false` and in `cssom.blocked[]`; extraction continues. **That sheet's breakpoints, `@font-face`, and keyframes are simply absent** — backfill (below). |
| Legacy `rule.type` | returns `0` for `@layer`, `@container`, `@property`, `@scope`, `@starting-style` | dispatch on `rule.constructor.name` — `CSSStyleRule`, `CSSMediaRule`, `CSSSupportsRule`, `CSSContainerRule`, `CSSLayerBlockRule`, `CSSLayerStatementRule`, `CSSScopeRule`, `CSSStartingStyleRule`, `CSSPropertyRule`, `CSSFontFaceRule`, `CSSKeyframesRule`, `CSSImportRule` — and recurse into anything exposing `.cssRules`. `cssom.ruleCounts` is keyed by that name, so an unexpected class shows up there |
| CSS Nesting | `CSSStyleRule.cssRules` is non-empty; nested selectors contain `&`, which `el.matches()` rejects | recursion does not treat style rules as leaves; `&` selectors are excluded from the specified-value index and raise `cssom:nested-selector-not-indexed` |
| `@import` | `.styleSheet` may be `null` (still loading) or throw | each touch wrapped; recursion continues |
| Shadow roots | constructed sheets never appear in `document.styleSheets`; closed roots are unreachable from page script | `adoptedCount`, `shadowRootsOpen`, and `shadowRootsSuspectedClosed` are reported. A closed root needs CDP `DOM.getDocument({pierce:true})`, which this MCP does not expose — say so in the report. |

Media/container/supports/layer context is threaded down the recursion, so `patterns.breakpoints[]` sees conditions
buried inside `@layer` + `@supports`, and `CSSPropertyRule` lands in `color.registered[]` — animatable-token
registrations, without which the clone's transitions on custom properties silently do nothing.

**Backfill a blocked sheet.** Do it before the merge; a missing sheet is usually where the real breakpoint system lives.

```jsonc
{ "resourceTypes": ["stylesheet"], "pageSize": 100 }                  // cdp:list_network_requests — find it
{ "reqid": 41, "responseFilePath": ".clone/css/vendor-tokens.css" }   // cdp:get_network_request — .network-response
```

The saved file gets a forced `.network-response` extension; rename it to `.css`.
Then either `Grep` the text for `@media`/`@font-face` and hand-merge those rows into `patterns.breakpoints[]` /
`fonts.faces[]`, or — for full CSSOM fidelity including nesting and `@layer` order — re-walk it same-origin off a shim
page. Build the shim with `Bash` so the CSS text never enters your context (never interpolate a stylesheet into a
`function` string — a 200 KB sheet is a 200 KB tool call):

```bash
mv .clone/css/vendor-tokens.network-response .clone/css/vendor-tokens.css   # the extension is force-rewritten
{ printf '<!doctype html><meta charset=utf-8><base href="https://cdn.example.com/css/"><style media="not all">\n';
  cat .clone/css/vendor-tokens.css; printf '\n</style>\n'; } > .clone/css/vendor-tokens.html
# cdp:new_page {"url":"file:///abs/project/.clone/css/vendor-tokens.html"} -> install -> all() -> cdp:close_page
```

Inline `<style>` is always readable, so nothing is blocked on the shim. `<base>` makes relative `url(...)` inside the
sheet resolve against the **real** origin, so recovered `fonts.faces[].srcUrls` are downloadable. `media="not all"`
keeps the rules out of the shim's live cascade while leaving the CSSOM fully readable — `readSheet` threads the
sheet-level media condition into the specified-value index, so those declarations contribute breakpoints, `@font-face`,
keyframes and custom properties without being mistaken for authored values that currently apply. Take only the
`cssom`/`patterns`/`fonts`/`color.vars` blocks, and log the recovery in `warnings[]`.

## 4. Used values are not authored values

`getComputedStyle` returns **resolved/used** values. The extractor keeps a parallel specified-value index off the CSSOM
(`specifiedFor`) — the only route back to authored intent. It orders by source position, not by the specificity cascade:
right ~95% of the time, wrong when a low-specificity later rule shadows a high-specificity earlier one. Sanity-check by
confirming the recovered value reproduces the measured geometry; otherwise fall back to shape inference.

| Authored | Computed gives you | Field that recovers it | Pixels can't |
|---|---|---|---|
| `repeat(3, minmax(0,1fr))` | `301.328px 301.328px 301.328px` | `space.grids[].authoredColumns` | a 3-up grid and three fixed columns render identically at one width |
| `clamp(2.25rem, 1.2rem + 4.2vw, 4.5rem)` | one number | `type.fluid[]` (§7) | one screenshot is one viewport |
| `right:24px; bottom:24px` on a FAB | `top:820px; right:24px; bottom:24px; left:1360px` | `topology.fixed[].authoredInset` | the corner offset and an absolute coordinate look the same until the window resizes |
| `max-width: 68ch` | `680px` here, `476px` there | `space.containers.authoredMaxWidths[]` | two different px values that are one authored decision |
| `currentColor`, `color-mix()`, `var()` | the resolved color | `cssom.customPropDeclarations[]`, `color.vars[]` | the indirection is invisible even at full zoom |
| `transparent` | `rgba(0, 0, 0, 0)` | — (know it is alpha-0 black, not the surface) | transparent and the surface color are the same pixels |
| `line-height: normal` | the literal string `normal` | `type.roles[].lhNormal` + metric-derived `lineHeightPx` | `parseFloat('normal')` is `NaN`, and a naive extractor ships `line-height: 0` |

`grid-template-areas` **is** preserved as authored strings in the computed value — when present it is the
highest-fidelity signal in the whole grid story; take it verbatim.

## 5. Color

Emit the **raw computed string** as the token value; RGBA/OKLab exist only for clustering. `oklch()`, `lab()`, `lch()`,
`hwb()`, and `color(display-p3 …)` are preserved in their own space by Chrome's serializer — re-quantizing them to hex
throws away gamut. Legacy sRGB collapses to `rgb()`/`rgba()`.

Weighting, because occurrence-counting ranks a 1px border used 60× above the hero background used once:

| Role | Weight | Pixels can't |
|---|---|---|
| `background-color`, gradient stops | `rect.w × rect.h`, clamped | tell a token from a one-off tint |
| `color`, SVG `fill`/`stroke` | `ownTextChars × fontSizePx` (ink-area proxy) | separate ink from surface at all |
| `border-*-color`, `outline-color` | `perimeter × borderWidth` | resolve a 1px hairline's exact alpha |

**The `fill` trap:** `fill`/`stroke` are inherited and `fill`'s initial value is black, so an unguarded harvest scores
`rgb(0,0,0)` on every `<div>` and it outranks the brand color by two orders of magnitude (measured: 11.5M vs 12.7K).
The extractor gates them on `el.namespaceURI === 'http://www.w3.org/2000/svg'`.

Clustering: greedy, weight-descending, in OKLab — merge when `√(ΔL²+Δa²+Δb²) ≤ 0.025` and `|Δα| ≤ 0.04`. The heaviest
member is the representative (`palette[].value`); the rest stay in `members[]` so `references/foundation.md` can decide
"these four near-blacks are one token". `palette[].roles[]` carries where the color was painted (`text`, `background`,
`border`, `fill`, `gradient-stop`, `shadow`) — that is what makes role assignment measured rather than guessed, and
`byRole` keeps the unclustered per-role rankings for cross-checking.

Custom properties: `getPropertyValue('--x')` returns the computed **token stream** — `var()` substituted but not
type-computed, so `--space: 1rem` stays `"1rem"`. Exactly what token emission wants. Harvest from both directions:
iterate the `:root` `CSSStyleDeclaration` (`scope: ":root"`), **and** every `--*` declaration seen while walking the
CSSOM, each read back off `:root` and off the first element matching its declaring selector — that is how
`[data-theme="dark"]`, `.card`, and per-component scopes survive (`scope` then holds the selector, suffixed
`(declared)` when nothing on the page currently resolves it).

Dark mode runs whenever `color.darkMode` shows signal: a `prefers-color-scheme` entry in `patterns.mediaFeatures`, a
non-`normal` `color-scheme` on `:root`, or a `.dark` / `[data-theme]` / `[data-mode]` toggle selector in the CSSOM.

```jsonc
{ "colorScheme": "dark" }                                                  // cdp:emulate — restate every option you want kept
{ "function": "() => window.__clone.foundation.all()", "filePath": ".clone/raw/foundation-dark.json" }
{ "colorScheme": "auto" }                                                  // cdp:emulate — always restore
```

**`emulate {colorScheme}` can reload the page, so run the dark pass last** — after motion, after the prewarm, after
the fluid sweep. A reload takes `window.__clone` *and* `window.__cloneHooks` with it, and `initScript` applies to one
navigation only. Check before you trust the dump: `() => ({ v: window.__clone && window.__clone.v, hooks:
!!window.__cloneHooks })`. If `v` is null, recover per `references/motion.md` M1 — re-`navigate_page` with the HOOKS
IIFE, re-install all four payloads, re-prewarm — and if hook data was lost before it was drained, say so in
`motion.json.warnings` rather than re-deriving it from a post-reveal page.

`emulate` resets every option you omit. If the page themes by class rather than by media query, also flip it in-page
(`() => { document.documentElement.classList.add('dark'); return document.documentElement.className; }`), re-run
`vars()`, and undo it. `emulate` exposes **colorScheme only** — there is no `prefers-reduced-motion` or `forced-colors`
emulation; those come out of `patterns.mediaFeatures[]` as authored rules and ship `authored-not-verified`.

## 6. Fonts

Three sources, joined on `(family, weight, style, unicodeRange)` into `fonts.faces[]` with `source:"cssom"|"fontfaceset"|"both"`:

| Source | Gives | Missing |
|---|---|---|
| CSSOM `@font-face` | `src` URLs (resolved against the **sheet's** href, not the document's), `format()`/`tech()`, `unicode-range`, `font-display`, `size-adjust`, ascent/descent overrides | load status |
| `document.fonts` (FontFaceSet, iterated with `forEach`) | `status` (`unloaded\|loading\|loaded\|error`), plus the same descriptors | **never exposes `src`** |
| `cdp:list_network_requests {resourceTypes:["font"]}` | the files actually fetched after `unicode-range` pruning | which declaration they belong to |

`status:"loaded"` ∧ in the census ⇒ mirror it. In CSSOM but never fetched ⇒ declared-only, keep the declaration and skip
the download. In the census but not in CSSOM ⇒ injected by JS (Typekit, the Google Fonts CSS API, `new FontFace()`);
`fonts.networkFontUrls[]` plus the census is the mirror list either way (`references/assets.md`).

**Resolved family.** `getComputedStyle(el).fontFamily` returns the authored *stack* verbatim and tells you nothing about
which entry won; Chrome's per-node used-font data (CDP `CSS.getPlatformFontsForNode`) is not exposed here. The extractor
uses the canvas width differential: measure `mmmMMMWWWiiill@#$%0Og8B` at `72px "F", <generic>` against `72px <generic>`
for all three generics; a >0.5px delta means `F` resolved. Walk the stack left→right, first available entry wins →
`fonts.usage[].resolvedFamily`, and the same value lands on every `fingerprint[]` row. `document.fonts.check()` is
recorded as `checkPasses` — a secondary signal only (it answers "renderable", true for system fonts too), so a
disagreement between the two is visible rather than silently averaged.

`fonts.synthesisRisk[]` names every `family @weight` a page requests without a matching non-variable face: the browser
faux-bolds it. Pixels can't tell a real 600 from a synthesized one, and reproducing it needs either the missing file or
a deliberate `font-synthesis` decision. Variable-font **axis inventories** live in the binary's `fvar` table, not the
DOM: the DOM has only the request (`font-variation-settings`, a `100 900` weight descriptor ⇒ `isVariable:true`). Read
axes offline from the mirrored woff2 if the rebuild needs them.

## 7. Type scale and the fluid sweep

`type.roles[]` is one row per distinct `role|size|weight|lhRatio|trackingEm|transform|family|style|variationSettings`
bucket, weighted by character count, kept only for elements that carry text of their own and either are a typographic
tag or differ in `font-size` from their parent (that is the definition of "carries type intent"). `type.scale[]` is the
deduped size ladder with char weight; `type.ratios[]` the adjacent-step ratios — a consistent 1.125/1.2/1.25/1.333
tells you the authored scale.

- `line-height: normal` is rebuilt from the font's own metrics (`measureText('Hxg')` →
  `fontBoundingBoxAscent + fontBoundingBoxDescent`, falling back to `1.2 × fontSize`), with `lhNormal:true` kept so the
  clone emits `normal` rather than a number that drifts when the font file changes.
- `trackingEm = letterSpacingPx / fontSizePx`, with `trackingNormal` flagged. Emitting `em` is what makes tracking
  survive a different base size.

**Fluid sweep.** Seven widths, not five, **at every profile** — this is the one measurement `--profile cheap` does not
shrink, and `references/scaling.md`'s profile table marks it so. A typical `clamp()` is railed at both ends of a
5-width sweep, leaving only two interior samples — the bare minimum the solver needs; below four the rails are
invented, not measured (`references/responsive.md` R5). Install `scripts/probe-responsive.js` too (same install sentence,
`filePath: ".clone/raw/install-responsive.json"`), cache the probe set once at the primary width, then sweep ascending:

```
() => window.__clone.responsive.probes()                     -> .clone/raw/probes.json     (once, at 1440)
for W in 360 480 768 1024 1200 1440 1920:
  cdp:resize_page {width: W, height: 900}
  () => window.__clone.responsive.measure('resize')          -> .clone/raw/fluid-<W>.json
```

Use `resize_page`, not `emulate {viewport}` — the sweep only needs layout width, and `emulate` also changes DPR/mobile/
touch and can reload. Then build the sample table from the dumps and solve in one call, interpolating the literal into
the source (`args` takes uids only):

```bash
node - <<'JS'   # -> the TABLE literal for the next call
const fs=require('fs'),P=['font-size','line-height','letter-spacing','padding-top','padding-bottom','gap','max-width','border-radius'];
const T={};for(const w of [360,480,768,1024,1200,1440,1920]){
  const t=fs.readFileSync(`.clone/raw/fluid-${w}.json`,'utf8');
  const d=JSON.parse(t.slice(t.indexOf('{'),t.lastIndexOf('}')+1));
  for(const e of d.els||[]) if(e.present&&e.st) for(const p of P){
    const v=parseFloat(e.st[p]); if(!Number.isFinite(v)) continue;
    (T[e.sel+'|'+p]=T[e.sel+'|'+p]||[]).push([w,+v.toFixed(3)]);}}
const rows=Object.entries(T).filter(([,s])=>s.length>=4&&new Set(s.map(x=>x[1])).size>1);
fs.writeFileSync('.clone/raw/fluid-table.json',JSON.stringify(rows));
console.log(rows.length+' varying (path,prop) pairs');
JS
```

```jsonc
// paste the contents of .clone/raw/fluid-table.json in place of TABLE
{ "function": "() => Object.fromEntries(TABLE.map(([k,s]) => [k, window.__clone.responsive.fitFluid(s)]))",
  "filePath": ".clone/raw/fluid-solved.json" }
```

Each value is a `type.fluid[]` record: `kind` (`fixed|clamp|linear|stepped|insufficient`), `minPx`/`maxPx`, `aPx`/`bVw`,
`r2`, `points`, `railedLow`/`railedHigh`, `css`, `cssRem`, `steps[]`. The solver, its interior fit and rail verification,
and what `minIsObservedOnly` means are in `references/responsive.md` — read them before trusting a rail. Two rules
matter here: `kind:"stepped"` with `points < 4` means "add sweep widths between the rails", not "it is stepped"; and a
non-monotonic series means a container query or JS drives the value, so re-probe against the `container-type` ancestor
(`patterns.containerTypes[]`) rather than the viewport.

## 8. Space, containers, grid

`space.baseUnitPx` is scored, not GCD'd — sub-pixel layout and 1.5px borders defeat GCD. Coverage of each candidate
`u ∈ {2,3,4,5,6,8,10,12,16}` over authored `padding-*`/`margin-*`/`gap` values in `(0,400]`; the **largest** `u` clearing
0.85 wins, and its coverage ships as `baseUnitConfidence` (a 4px-grid site scores `4:0.97, 8:0.71`). Rect dimensions are
deliberately excluded — those are layout outcomes, not spacing decisions. If confidence is under 0.85 there is no grid:
emit the observed scale and say so, do not round values to invent one.

Containers get three independent recoveries because each fails alone: clustered computed `max-width` (`maxWidths`,
resolved and therefore noisy), authored declarations off the CSSOM (`authoredMaxWidths`, where `68ch`, `min(100%,72rem)`
and `--container-lg` survive intact), and a weighted `rect.width` histogram over block-level elements whose parent is
≥24px wider (`contentWidths` — the dominant mode is the content column), all tagged with `measuredAtPx`. Grids carry
`computedColumns` **and** `authoredColumns`/`authoredRows`/`areas` plus `gapPx`/`autoFlow`/`itemCount`; flex containers
carry direction/wrap/justify/align/gap/childCount. `gapPx` is a single number per the schema (column gap, falling back
to row gap), so a container whose `row-gap` and `column-gap` differ is only fully described by its per-section
`layout.gap` string in `sections.json` — read that before emitting a `gap` shorthand.

`all()` rebuilds its CSSOM accumulators on every invocation, so re-running it on the same page — which the dark pass
and every extra `--viewport` do — never double-counts rules, faces or keyframes. Only `cdp:navigate_page` requires a
re-install.

## 9. Topology and patterns

| Block | What it is for | Pixels can't |
|---|---|---|
| `topology.tree[]` | ≤300 structural nodes with `path`, `rect`, `isSection`, `fullBleed` — the input to segmentation | see nesting, only adjacency |
| `stackingContexts[]` + `zLadder[]` | every context with its reason list (`opacity<1`, `transform`, `filter`, `contain`, flex/grid child + z-index, `view-transition-name`, …) and the unique numeric `z-index` ladder with owner paths | paint order is invisible until a dropdown clips; z-index leaves no pixels |
| `sticky[]` / `fixed[]` | used **and** authored insets, `z-index`, `backdrop-filter`, background | a pinned header at scrollY 0 looks static |
| `scrollers[]` + `scroll.timelines[]` | overflowing containers with axis/`scroll-snap-type`/behaviour, and `animation-timeline`/`view-timeline-name`/`animation-range` (native scroll-driven animation, trivially reproducible) | a snap carousel at slide 1 is a static row; motion is not in a still |
| `effects[]` | frequency map of `transform`/`filter`/`backdrop-filter`/`mix-blend-mode`/`clip-path`/`mask-image`/`contain`/… — value + painted-area weight, **no path**: this answers "does this page use glass" and nothing more | a blurred glass panel and a flat translucent one look alike |
| `effectNodes[]` | the five unbuildable-without-a-path ones — `filter`, `backdropFilter`, `mixBlendMode`, `clipPath`, `maskImage` — as `{path, prop, value, area}`, cap **80**, `warnings:["cap:effectNodes"]` when it overflows. Copy `value` verbatim into the owning section's `spec.effects[]` (`references/agent-brief.md` §1); it is the only channel these properties have into a brief, and the geometry gate cannot see them | a glass card shipped as a flat `bg-white/70`, a blend-mode headline shipped as plain text, a clipped shape shipped as a rectangle |
| `pseudos[]` | `::before`/`::after` with `content` and the decls that position them | pseudo-element decoration reads as markup |

Two Chrome-specific noise sources are already filtered and must stay filtered: `view-transition-name: root`, which
Chrome sets on `:root` itself, and identity transform matrices left behind by `animation-fill-mode: both`.

`patterns` holds the weight-ranked global sets: `radii` (per corner and as a set — `9999px` and `50%` are different
intents, do not normalize), `shadows` (Chrome serializes **color first**: `rgba(0,0,0,.1) 0px 1px 2px 0px`, multiple
shadows split at top-level commas), `borders`/`outlines` (only sides with width > 0 ∧ style ≠ none), `transitions` (four
parallel comma lists zipped by index, shortest cycling per spec), `animations` + `keyframes` (each `CSSKeyframesRule`
name with every `keyText` + `cssText`), `easings`/`durations` (keep a `linear()` point list verbatim — it is a spring
approximation and rounding it kills the feel), `mediaFeatures`, `containerQueries`/`containerTypes`, `supports` (each
re-tested with `CSS.supports`, so you know which branch is live), and `layers` (the authored `@layer` order — the
cascade architecture). `patterns.keyframes[]` overlaps `motion.json` by design: static census here, behavioural truth
and rebuild recipes in `references/motion.md`.

Breakpoints come from every `CSSMediaRule.conditionText`, both syntaxes (`(min-width: 768px)` and
`(48rem <= width < 64rem)`), with `em`/`rem` converted at **16** — a media-query `em` is always relative to the initial
font size, never to `:root`, one of the few places where 16 is guaranteed. The 4–6 highest-count values are the real
system; one-offs are patches.

## 10. `fingerprint[]`

The numeric verification contract, and the reason to run a byte-identical extractor against both sides. ≤80 rows
matching `header,nav,main,footer,section,h1,h2,h3,button,a[class],[class*=hero],[class*=cta]`, each with `path`, `tag`,
`rect` (document coords — measure at `scrollY === 0`), `fontFamily`, `resolvedFamily`, `fontSizePx`, `fontWeight`,
`lineHeightPx`, `color`, `bg`, `textLen`, `textHead`, `textWidthPx` (a `Range` around the element's contents).
`textWidthPx` is the load-bearing field: right family name, wrong subset/features/tracking, and it is the only number
that moves. Re-run with `() => window.__clone.foundation.fingerprint()`; join key and tolerances are in
`references/assembly.md`.

## 11. Merge

Reproducible, because verification re-runs it. `all()` leaves exactly two holes: `type.fluid` and
`color.darkMode.tokenDiff`.

```bash
node - <<'JS'
const fs=require('fs'), R='.clone/raw/', J=p=>{const t=fs.readFileSync(p,'utf8');return JSON.parse(t.slice(t.indexOf('{'),t.lastIndexOf('}')+1));};
const f=J(R+'foundation-1440.json');                                    // primary width in the filename
const solved=fs.existsSync(R+'fluid-solved.json')?J(R+'fluid-solved.json'):{};
f.type.fluid=Object.entries(solved).map(([k,v])=>{const i=k.lastIndexOf('|');
  const {samples,crossoverLow,crossoverHigh,...keep}=v; return {path:k.slice(0,i),prop:k.slice(i+1),...keep};})
  .filter(e=>e.kind&&e.kind!=='insufficient');
if(fs.existsSync(R+'foundation-dark.json')){const d=J(R+'foundation-dark.json');
  // the dark dump is all() ({color:{vars}}) or the cheap vars() re-read ({vars}) — accept either
  const m=new Map((d.color?d.color.vars:d.vars||[]).map(v=>[v.name,v.value]));
  f.color.darkMode.tokenDiff=f.color.vars.filter(v=>m.has(v.name)&&m.get(v.name)!==v.value)
    .map(v=>({name:v.name,light:v.value,dark:m.get(v.name)}));
  // schema D1's darkMode object is closed; extra evidence goes in the open `counts` bag, never in darkMode
  if(d.fingerprint) f.counts.darkFingerprintDeltas=d.fingerprint.filter((r,i)=>f.fingerprint[i]&&(r.color!==f.fingerprint[i].color||r.bg!==f.fingerprint[i].bg)).length;}
fs.writeFileSync('.clone/foundation.json.tmp',JSON.stringify(f,null,1));
fs.renameSync('.clone/foundation.json.tmp','.clone/foundation.json');   // atomic: a half-written merge fails --resume
console.log(JSON.stringify({schema:f.schema,fluid:f.type.fluid.length,tokenDiff:f.color.darkMode.tokenDiff.length,
  faces:f.fonts.faces.length,palette:f.color.palette.length,breakpoints:f.patterns.breakpoints.length,
  blocked:f.cssom.blocked.length,fingerprint:f.fingerprint.length,baseUnit:f.space.baseUnitPx,warnings:f.warnings.length}));
JS
```

Everything else comes from the primary-width dump unchanged — the dark and swept dumps stay in `raw/` as evidence and are
never merged field-by-field, because a dark `rect` or a 360px `fontSizePx` in the primary record poisons every
downstream token. Fluid entries whose `path` no longer resolves are dropped, not invented.

## 12. Exit gate for this pass

SKILL.md's step-2 gate covers the three JSONs step 2 produces; these are the `foundation.json` rows (`motion.json` and
`responsive.json` are gated in `references/motion.md` and `references/responsive.md`; `sections.json` is step 4a's,
gated in `references/sectioning.md`).

- [ ] `.clone/foundation.json` exists and `schema === "clone-site/foundation@1"`.
- [ ] `fonts.faces[]` non-empty whenever the page uses webfonts (`fonts.usage[].isWebfont` true anywhere), and every
      such family has ≥1 entry in `fonts.networkFontUrls[]` or a census font row.
- [ ] `patterns.breakpoints[]` populated. Empty on a responsive-looking page ⇒ a blocked sheet, so §3 first.
- [ ] `cssom.blocked[]` empty, or every entry has a file in `.clone/css/` and its rules folded in.
- [ ] `fingerprint[]` non-empty — it is the numeric target verification diffs against.
- [ ] `space.baseUnitPx > 1` with `baseUnitConfidence ≥ 0.85`, or a `warnings[]`/report line saying there is no grid.
- [ ] `warnings[]` read once: `cap:elements`/`cap:rules` means a cap was hit and the ranked lists are partial — note it
      in `CLONE-REPORT.md` rather than pretending completeness.

## 13. Honest limits

| Limit | What to do |
|---|---|
| Per-node used font is not exposed to page JS | canvas differential + `document.fonts.check`; both recorded so disagreement is visible |
| Variable-font axis inventory is in the binary | read `fvar` offline from the mirrored woff2 |
| Specified-value index orders by source position, not by specificity | verify against geometry, fall back to track-shape inference |
| Closed shadow roots | counted only (`shadowRootsSuspectedClosed`); needs CDP `pierce`, unavailable here |
| 8-bit RGBA clustering clips wide gamut | clustering only — the emitted token is always the raw computed string |
| Hover/focus/`:active` and scroll-dependent state are not in this pass | `references/motion.md` (its sweep re-reads the `sticky[]`/`fixed[]` paths); `:active` is unverifiable through this MCP |
| JS-driven layout (masonry, Swiper) yields px no CSS produced; content behind consent or auth never renders | `stack.animation[]` flags the first — treat those sections as library-backed; dismiss the dialog before pass 2 (`references/troubleshooting.md`) for the second |
