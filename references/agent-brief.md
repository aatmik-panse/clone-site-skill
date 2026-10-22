# Agent brief — the exact contract handed to one `clone-section` agent

One directory per section is the **entire** input to the agent. If a fact is not in that directory, the agent does
not know it and will invent something. Nothing is inherited.

```
.clone/sections/01-hero/
├── spec.json        # measured truth (§1)             orchestrator writes
├── content.md       # verbatim text, DOM order (§2)   orchestrator writes
├── PROMPT.md        # the literal task text (§3)      orchestrator writes
├── orig-w1440.webp  ├── orig-w390.webp   # the visual targets the agent reads   orchestrator writes
├── orig-w1440.png   ├── orig-w390.png    # same shots, PNG — visual-diff.mjs input only, never handed to an agent
├── report.md        # ← agent writes (§7)
└── requests.json    # ← agent writes (§6); it never edits a shared file itself
```

Dispatch: one `Task` call per section, `subagent_type: "clone-section"`, prompt = the **contents** of that section's
`PROMPT.md` (not its path). Escalate to `model: opus` when `carousel != null`, `counts.forms > 0`, or the motion slice
has any `scrollLinked` entry. Wave composition and the ownership map: `references/sectioning.md`.

## 1. `spec.json` — the contract, filled in (hero, `01-hero`)

Every value came out of `.clone/sections.json`, `foundation.json`, `motion.json`, `responsive.json`, `assets.json`.
Nothing is estimated: if you could not measure it, it is absent, not guessed. Width-keyed maps use the integer width
as a string — never `desktop`/`mobile`.

```jsonc
{
  "contract": 1, "id": "01-hero", "pageId": "home", "order": 1, "role": "section",
  "label": "Ship signal, not glue code", "sourceSelector": "#hero", "extraSelectors": [".hero-glow"],
  "writeFile": "components/sections/Hero.tsx",          // the ONE file this agent owns
  "exportName": "Hero", "exportKind": "named function component, no default export",
  "props": "none — content is inlined verbatim", "clientComponent": false,
  "usedBy": "app/page.tsx — the orchestrator wires it; do not edit that file",
  "primaryWidth": 1440, "widths": [1440, 390],
  "box": { "1440": { "x": 0, "y": 72, "w": 1440, "h": 812 }, "390": { "x": 0, "y": 56, "w": 390, "h": 946 } },
  "container": { "maxWidth": "1152px", "background": "--color-surface",
                 "paddingInline": { "1440": "48px", "390": "20px" }, "paddingBlock": { "1440": "112px 96px", "390": "72px 64px" },
                 "backgroundImage": "radial-gradient(60% 50% at 50% 0%, oklch(0.96 0.04 264), oklch(1 0 0))",
                 "note": "the gradient is the .hero-glow child, not a background on the section root" },
  "layout": { "1440": { "display": "grid", "gridTemplateColumns": "minmax(0,560px) minmax(0,1fr)", "gap": "64px", "alignItems": "center", "minHeight": "812px" },
              "390": { "display": "flex", "flexDirection": "column", "gap": "40px", "alignItems": "stretch" },
              "breakpointsObserved": [768, 1024], "note": "two columns at ≥1024, single column below; measured DOM order is unchanged" },
  "typography": {
    "eyebrow": { "text": "Now in general availability", "token": "--text-xs", "weight": 600, "letterSpacing": "0.08em", "textTransform": "uppercase", "color": "--color-accent" },
    "h1": { "text": "Ship signal, not glue code", "token": "--text-6xl", "weight": 600, "lineHeight": "1.05",
            "letterSpacing": "-0.03em", "color": "--color-ink", "family": "--font-display",
            "balance": "text-wrap: balance (computed on the original)",
            "fluid": "44px @390 → 72px @1440, solved clamp(2.75rem, 1.06rem + 2.36vw, 4.5rem), r²=0.998" },
    "lede": { "text": "…see content.md", "token": "--text-lg", "weight": 400, "lineHeight": "1.6", "color": "--color-ink-muted", "maxWidth": "34rem" },
    "ctaPrimary": { "token": "--text-base", "weight": 550, "color": "--color-accent-contrast", "background": "--color-accent",
                    "padding": "12px 22px", "radius": "--radius-full", "shadow": "--shadow-sm" },
    "ctaSecondary": { "token": "--text-base", "weight": 500, "color": "--color-ink", "background": "transparent", "padding": "12px 18px", "radius": "--radius-full", "border": "1px solid --color-border-subtle" } },
  "probes": [
    { "name": "root",    "selector": "#hero",          "box": { "x": 0,   "y": 72,  "w": 1440, "h": 812 } },
    { "name": "eyebrow", "selector": "#hero .eyebrow", "box": { "x": 144, "y": 184, "w": 232,  "h": 18 } },
    { "name": "heading", "selector": "#hero h1",       "box": { "x": 144, "y": 218, "w": 560,  "h": 152 } },
    { "name": "body",    "selector": "#hero p",        "box": { "x": 144, "y": 394, "w": 544,  "h": 58 } },
    { "name": "cta",     "selector": "#hero a.btn",    "box": { "x": 144, "y": 484, "w": 178,  "h": 46 } },
    { "name": "media",   "selector": "#hero img",      "box": { "x": 768, "y": 176, "w": 528,  "h": 396 } }],
  "probeRule": "Verification re-measures these six boxes in your output and reports every delta. Match the geometry, not just the look.",
  "tokensAllowed": {
    "--color-ink": "oklch(0.19 0.02 265)", "--color-ink-muted": "oklch(0.52 0.02 265)", "--color-surface": "oklch(1 0 0)",
    "--color-accent": "oklch(0.62 0.19 265)", "--color-accent-soft": "oklch(0.95 0.03 265)", "--color-accent-contrast": "oklch(1 0 0)",
    "--color-border-subtle": "oklch(0.92 0.006 265)", "--radius-full": "9999px", "--spacing": "4px", "--container-6xl": "1152px",
    "--font-display": "\"Söhne Breit\", system-ui, sans-serif", "--font-sans": "\"Söhne\", system-ui, sans-serif",
    "--text-xs": "12px", "--text-base": "16px", "--text-lg": "18px", "--text-6xl": "clamp(2.75rem, 1.06rem + 2.36vw, 4.5rem)",
    "--shadow-sm": "0 1px 2px oklch(0.19 0.02 265 / 0.06)", "--ease-out-quart": "cubic-bezier(0.25,1,0.5,1)" },
  "tokenRule": "Every color, font, size, radius, shadow, gap and easing comes from this map, written as the Tailwind v4 utility @theme generates (text-ink-muted, bg-surface, font-display, rounded-full, shadow-sm, gap-16, ease-out-quart). ZERO literal hex/rgb/hsl/oklch in your file. --spacing is 4px, so gap-16 = 64px and p-12 = 48px — do the arithmetic, never eyeball it. Missing a value? File it in requests.json, use the nearest existing token meanwhile, and say so in report.md.",
  "assets": [
    { "role": "shot", "ref": "/assets/img/hero-dash-2x.webp", "source": "https://cdn.acme.com/hero/dash@2x.webp", "w": 1056, "h": 792,
      "renderedW": 528, "renderedH": 396, "loading": "eager", "fetchPriority": "high", "alt": "see content.md",
      "sizes": "(min-width:1024px) 528px, 100vw" },
    { "role": "logo", "ref": "/assets/svg/logo-stripe.svg", "source": "https://cdn.acme.com/logos/stripe.svg", "w": 84, "h": 20 }],
  "assetRule": "Assets are already mirrored byte-for-byte. Reference the `ref` path exactly, character for character. `source` is provenance only — a remote URL must never appear in your file. Never fetch, never rename, never invent a filename, never substitute an emoji or a hand-drawn SVG for a mirrored file.",
  "motion": {
    "entrance": [{ "target": "#hero > .stack > *", "count": 4, "mechanism": "css-transition + IO class",
                   "from": { "opacity": 0, "transform": "translateY(14px)" }, "to": { "opacity": 1, "transform": "none" },
                   "durationMs": 520, "staggerMs": 70, "delayMs": 0, "easing": "cubic-bezier(0.25,1,0.5,1)",
                   "trigger": "in-view once, rootMargin -12%", "evidence": "motion-anims-load.json#animationsAtLoad[3]" }],
    "hover": [{ "target": "a.btn", "transition": "160ms cubic-bezier(0.25,1,0.5,1)", "route": "both",
                "delta": { "backgroundColor": "oklch(0.56 0.19 265)", "transform": "translateY(-1px)" },
                "evidence": "motion-states-cssom.json#interactive[11]" }],
    "scrollLinked": [], "reducedMotion": { "authored": true, "clonePlan": "no transform, opacity only, 160ms" },
    "attrs": [{ "path": "div > header > h2", "attrs": { "data-anim": "heading", "data-anim-delay": "0.1", "data-split-lines": "" },
                "evidence": "motion-attrs.json#elements[14]" }],
    "attrsRule": "When the origin drives motion declaratively, `attrs[]` is the site's OWN vocabulary, harvested per section (references/motion-source.md §S6). Copy each attribute onto the matching element and let the shared runtime animate it — do NOT hand-roll per-section motion. `path` is relative to your section root. A shared runtime plus declared attributes is how the origin is built; N sections each inventing their own reveal is how a clone drifts.",
    "rule": "Content is visible by default; the reveal enhances it. Never ship opacity:0 with no fallback — a headless render or a background tab leaves the section blank." },
  "responsive": {
    "layout": [{ "property": "grid-template-columns", "evidence": "vp-390.json#deltas[7]",
                 "byViewport": { "1440": "minmax(0,560px) minmax(0,1fr)", "390": "none" },
                 "solved": "grid-cols-1 lg:grid-cols-[minmax(0,560px)_minmax(0,1fr)]" }],
    "fluid": [{ "target": "#hero h1", "property": "font-size", "fit": { "kind": "clamp", "r2": 0.998 },
                "samples": [[390, 44], [768, 51.2], [1024, 57.2], [1440, 72]], "solved": "text-6xl (the token already clamps)" }],
    "breakpointOnly": [{ "target": "#hero .trust-strip", "existsAt": [1440], "hiddenAt": [390],
                         "note": "display:none below 768 — reproduce the breakpoint, do not delete the markup" }],
    "overflowRisks": [{ "target": "#hero img", "at": 360, "note": "fixed 528px overflows; original uses max-width:100%" }] },
  "effects": [
    { "target": "#hero .glass-card", "prop": "backdrop-filter", "value": "blur(18px) saturate(1.4)", "note": "the card is oklch(1 0 0 / 0.62) over the gradient — the blur is what makes it read as glass",
      "evidence": "foundation.json#topology.effectNodes[3]" },
    { "target": "#hero h1 > em", "prop": "mix-blend-mode", "value": "difference", "evidence": "foundation.json#topology.effectNodes[7]" }],
  "effectRule": "Copy `value` character for character — `backdrop-filter`, `mix-blend-mode`, `clip-path`, `mask-image` and `filter` are invisible to the geometry probes and survive a pixel diff at a low score, so nothing downstream will catch a wrong one. A backdrop-filter also needs its own stacking context and a translucent background to do anything. If the property has no token, an arbitrary value is correct here and does not count against the arbitrary-value budget; list it in report.md.",
  "a11y": { "landmark": "section", "labelledBy": "h1#hero-title", "headingLevel": 1,
            "decorative": [".hero-glow is aria-hidden in the original"], "focusVisible": "inherits the global :focus-visible ring — do not override" },
  "dependencies": { "allowed": ["react", "next/image", "next/link"],
                    "forbidden": "anything not in this list — file a requests.json entry instead. You cannot install packages; you have no Bash." },
  "captures": { "1440": "orig-w1440.webp", "390": "orig-w390.webp" }, "content": "content.md",
  "gates": { "source": "references/assembly.md § per-section gates", "copiedIntoPrompt": true }   // numbers live in one file only
}
```

| Field | Source | Trap it avoids |
|---|---|---|
| `box`, `layout`, `probes` | the `sections.json` entry, width-keyed | agent guesses padding off a screenshot |
| `typography.*.fluid` | `foundation.json.type.fluid`, `responsive.json.bySection[].fluid` | a hard-coded 72px heading that never shrinks |
| `tokensAllowed` | `foundation.json` values, filtered to what this section actually paints | reaching for a token that does not exist, or hardcoding a hex |
| `assets` | `assets.json` rows whose `usedBy[]` contains this section id; `spec.assets[].ref` **is** `assets.json[].ref` (the public path markup writes), never `assets.json[].local` (staging-relative, `img/hero.avif`, which would ship as a broken relative src) | re-downloading, renaming, emoji substitution, a src that 404s on every route but `/` |
| `motion`, `responsive` | `motion.json.bySection[<id>]` / `responsive.json.bySection[<id>]` verbatim, `evidence` included | invented animation. An entry with no `evidence` is deleted before the brief is written |
| `dependencies.allowed` | what the detected stack already installs | `npm i` attempts, phantom imports |
| `captures` | the filenames actually written by `references/sectioning.md` §7 | an agent reading a path that does not exist and proceeding blind |
| `repeat`, `carousel`, `pinned`, `forms` | `sections.json` plus the audit in `references/sectioning.md` §4 | twelve hand-written cards; a pinned track split into four sections; a bare `<input>` where the original had `appearance:none` |
| `effects` | `foundation.json.topology.effectNodes[]` rows whose `path` is inside this section | a glass panel shipped flat — no probe and no diff score reliably catches it |

Keep a spec under ~200 lines. Longer means the section is too big — re-cut it (`references/sectioning.md` §3).

## 2. `content.md` — verbatim text in DOM order

From `window.__clone.sections.content("<selector>")`: one line per row, `key` straight from the payload (`item[<i>].`
prefix for repeat children), text in backticks so leading/trailing spaces and typographic characters survive review.
The cheapest fidelity control in the pipeline — the only thing that reliably stops an agent from "improving" the
copy, which is both a fidelity failure and a verbatim-mirror violation.

```md
<!-- 01-hero — verbatim. Copy character for character. No rewording, no "improved" copy,
     no straight-quote or em-dash substitutions, no Title Case fixes. -->
- eyebrow: `Now in general availability`
- h1: `Ship signal, not glue code`
- body: `Point a webhook at us and we handle retries, ordering, and dedupe — so your team ships product instead of plumbing.`
- link: `Start free` -> `/signup`
- link[1]: `Book a demo` -> `/demo`
- text: `No credit card. 10k events free every month.`   <!-- a bare text node still gets a row -->
- img: `The Acme dashboard showing live event throughput` (/assets/img/hero-dash-2x.webp)
```

## 3. `PROMPT.md` — the literal text the hero agent receives

```md
Build ONE section component. You are one of 6 agents working in parallel on the same clone.
You cannot see the live site, you have no browser, and you have no shell. Your inputs are complete.

READ FIRST, in this order (absolute paths):
  1. /abs/acme-clone/.clone/sections/01-hero/spec.json        <- measured ground truth. Obey it exactly.
  2. /abs/acme-clone/.clone/sections/01-hero/content.md       <- copy this text verbatim.
  3. /abs/acme-clone/.clone/sections/01-hero/orig-w1440.webp  <- the target at 1440
  4. /abs/acme-clone/.clone/sections/01-hero/orig-w390.webp   <- the target at 390
  5. /abs/acme-clone/app/globals.css                          <- READ ONLY. The token source.
  6. /abs/acme-clone/components/sections/_EXAMPLE.tsx         <- house style for a section component

WRITE EXACTLY ONE FILE: /abs/acme-clone/components/sections/Hero.tsx
  export function Hero() { … }        // named export, no default export

YOU MAY NOT TOUCH: app/globals.css, app/layout.tsx, app/page.tsx, tailwind/postcss config, package.json, tsconfig.json, public/**, components/shared/**, or any other file in components/sections/. Writing to them corrupts five other agents' work in this wave.

ALLOWED IMPORTS: react, next/image, next/link — nothing else. You cannot install packages.

NEED SOMETHING SHARED? Append to /abs/acme-clone/.clone/sections/01-hero/requests.json (create it if absent):
  {"requests":[{"kind":"token","name":"--color-accent-strong","value":"oklch(0.56 0.19 265)","why":"measured CTA hover bg, no existing token within ΔE 2"}]}
  kinds: token | dependency | shared-component | global-css. Then proceed with the nearest existing token and
  say so in report.md. The orchestrator applies requests centrally.

DEFINITION OF DONE — self-check every line, report each verdict in report.md:
  [ ] Compiles as TSX. No `any`. No unused imports. Named export only.
  [ ] Every user-visible string matches content.md exactly, including punctuation and casing.
  [ ] Zero literal colors: no #hex, no rgb(), no hsl(), no oklch() anywhere in the file.
  [ ] Every measurement in spec.json (padding, gap, font-size, weight, radius, max-width, min-height) appears as the corresponding token utility. Arbitrary values (p-[27px]) only where spec.json holds a value no token expresses, and each one is listed in report.md with its measured source.
  [ ] Geometry: the six probes in spec.probes land at their measured boxes at 1440.
  [ ] Responsive: matches orig-w390.webp at 390 using spec.layout.breakpointsObserved; nothing overflows horizontally at 360px; breakpointOnly targets are hidden, not deleted.
  [ ] Images use spec.assets[].ref with width/height set (no CLS) and the given loading/sizes/alt. No http:// or https:// anywhere in the file — not in a src, not in a comment.
  [ ] Motion per spec.motion, including the prefers-reduced-motion branch. Content visible by default.
  [ ] Semantic HTML: one <section>, heading level from spec.a11y, aria-labelledby wired, decorative nodes aria-hidden.
  [ ] No "use client" unless the section genuinely needs browser JS. If it does, line 1, and justify it.

GATES you will be measured against (verbatim from references/assembly.md § per-section gates — one source of truth for every number):
  {{GATE_TABLE}}

WRITE report.md, MAX 400 WORDS, exactly these five headings:
  ## Built <structure, why this layout mechanism> · ## Done-check <the checklist above, PASS / FAIL / N-A per line>
  ## Deviations <measured value, what you shipped, why> · ## Requests <count and kinds, or "none">
  ## Uncertainty <where you guessed, ranked worst first>
```

## 4. Second example — repeated card grid (`04-features`)

Only the fields that differ from §1. A 12-card section is one sub-component plus a data array, not 12 hand-written
cards: ~90 lines instead of ~600, and one place to fix when the card padding is wrong.

```jsonc
{
  "id": "04-features", "writeFile": "components/sections/Features.tsx", "exportName": "Features",
  "layout": { "1440": { "display": "grid", "gridTemplateColumns": "repeat(3, minmax(0,1fr))", "gap": "32px" },
              "768":  { "display": "grid", "gridTemplateColumns": "repeat(2, minmax(0,1fr))", "gap": "24px" },
              "390":  { "display": "grid", "gridTemplateColumns": "1fr", "gap": "24px" },
              "breakpointsObserved": [768, 1024] },
  "repeat": {
    "count": 6, "containerSelector": "#features .grid", "itemSelector": "#features .card",
    "columns": 3, "gap": "32px",
    "instruction": "Build ONE local Card sub-component in this same file plus a FEATURES const array of 6 objects, and map over it. Do not write six card literals. Do not export Card.",
    "item": { "padding": "28px", "radius": "--radius-lg", "border": "1px solid --color-border-subtle",
              "background": "--color-surface", "shadow": "--shadow-sm", "iconBox": { "size": "40px", "radius": "--radius-md", "background": "--color-accent-soft" },
              "title": { "token": "--text-lg", "weight": 600, "color": "--color-ink", "lineHeight": "1.3" },
              "body":  { "token": "--text-sm", "weight": 400, "color": "--color-ink-muted", "lineHeight": "1.6" },
              "hover": { "borderColor": "--color-border-strong", "transform": "translateY(-2px)",
                         "transition": "180ms cubic-bezier(0.25,1,0.5,1)", "evidence": "motion-states-live.json#hover[4]" } },
    "itemContentKeys": "content.md rows are prefixed item[0..5]. — keep the mapping exact; card 3's body belongs to card 3" },
  "probes": [{ "name": "root" }, { "name": "heading" }, { "name": "body" }, { "name": "repeat-item", "selector": "#features .card", "box": { "x": 144, "y": 2408, "w": 362, "h": 244 } }],
  "motion": { "reducedMotion": { "authored": true, "clonePlan": "opacity crossfade 160ms, no transform" },
              "entrance": [{ "target": "#features .card", "count": 6, "mechanism": "css-transition + IO class",
                             "from": { "opacity": 0, "transform": "translateY(12px)" }, "to": { "opacity": 1, "transform": "none" },
                             "durationMs": 420, "staggerMs": 60, "easing": "cubic-bezier(0.25,1,0.5,1)",
                             "trigger": "in-view once, rootMargin -15%", "evidence": "motion-anims-load.json#animationsAtLoad[9]" }] }
}
```

Prompt deltas — append to the DoD: `[ ] Six cards = 1 local Card component + a 6-item data array, zero duplicated
JSX.` · `[ ] Stagger is per-index (delay = index * 60ms), not one animation on the container.` · `[ ] The 2-up step at
768 exists (it was measured), not just 3-up and 1-up.`

## 5. Third example — pinned scroll section (`05-how-it-works`)

A tall track holding a short sticky stage, stepped by scroll progress. The track height **is** the scroll budget:
get it wrong and every step fires at the wrong scroll position, then the diff blames your styling. Always one agent,
one file, `model: opus`.

```jsonc
{
  "id": "05-how-it-works", "writeFile": "components/sections/HowItWorks.tsx", "exportName": "HowItWorks",
  "box": { "1440": { "x": 0, "y": 4160, "w": 1440, "h": 4500 }, "390": { "x": 0, "y": 5210, "w": 390, "h": 1980 } },
  "pinned": {
    "trackSelector": "#how", "trackHeight": { "1440": "4500px", "390": "auto" },
    "stageSelector": "#how .stage", "stageHeight": 812, "stagePosition": "sticky", "stageTop": "0px",
    "steps": 4, "stepSelector": "#how .step", "evidence": "section-audit-1440.json#audit[5]",
    "instruction": "Outer track = one element at the measured height with overflow:clip. Inner stage = position:sticky, top-0, height 100svh (measured 812px at 1440). Do NOT split this into four sections, do NOT animate the track itself, do NOT recompute the height from the number of steps. Below 768 the stage is position:static (measured) and the four steps stack in normal flow with no fixed track height." },
  "motion": {
    "scrollLinked": [
      { "target": "#how .stage-inner", "driver": "cover", "channel": "transform.translateX",
        "range": { "start": 0.08, "end": 0.92 }, "from": 0, "to": -2160, "unit": "px",
        "fit": { "kind": "linear", "r2": 0.997 }, "evidence": "motion-scroll.json#scrollLinked[2]",
        "rebuild": "animation-timeline: view(); animation-range: cover 8% cover 92%; @keyframes translateX(0) → translateX(-2160px)" },
      { "target": "#how .step", "driver": "cover", "channel": "opacity", "from": 0.35, "to": 1, "unit": "",
        "range": { "start": 0.08, "end": 0.92 }, "fit": { "kind": "stepped" },
        "evidence": "motion-scroll.json#scrollLinked[3]",
        "rebuild": "four discrete states on the same timeline: step i is active while progress ∈ [i/4,(i+1)/4); one @keyframes with 4 offsets" }],
    "reducedMotion": { "authored": false, "clonePlan": "@media (prefers-reduced-motion: reduce): drop the sticky track, render the four steps as a normal stacked list — same copy, same order" },
    "rule": "Reproduce the pin before you animate. Rebuild recipes and the CSS-first order (animation-timeline before any JS) are in references/motion.md — do not invent a scroll listener when a timeline was measured." }
}
```

Prompt deltas — append to the DoD: `[ ] Track height is exactly spec.pinned.trackHeight per width, not derived and
not rounded to a vh multiple.` · `[ ] Stage is position:sticky at the measured top and height, and stops being
sticky below 768.` · `[ ] Steps advance on scroll progress over the measured range — not a timer, not wheel
events.` · `[ ] Reduced-motion renders all four steps, stacked and readable, no sticky track.` · `[ ] Your visual
target is the tile set (orig-w1440-tile-00..02.webp) — judge composition per tile.`

## 5a. Fourth shape — a form (`08-signup`)

`counts.forms > 0` escalates the section to `model: opus`, and it is the shape with the most invisible surface: a
screenshot shows one resting field and none of the six states around it. `appearance: none` on a `<select>` is the
single most-missed declaration in clone work — miss it and the clone renders the OS widget instead of the design.

```jsonc
{
  "id": "08-signup", "writeFile": "components/sections/Signup.tsx", "exportName": "Signup",
  "form": {
    "formSelector": "#signup form", "method": "post", "action": null, "measuredAction": "/api/subscribe",
    "actionPolicy": "inert",             // inert (default) | kept — only the user may ask for `kept`
    "novalidate": false,
    "layout": { "1440": { "display": "grid", "gridTemplateColumns": "1fr auto", "gap": "12px" } },
    "fields": [
      { "name": "email", "type": "email", "required": true, "autocomplete": "email", "inputmode": "email",
        "placeholder": "you@company.com", "labelText": "Work email", "labelVisible": false, "ariaLabel": "Work email",
        "box": { "1440": { "w": 320, "h": 48 } }, "padding": "12px 16px", "radius": "--radius-md",
        "border": "1px solid --color-border-subtle", "background": "--color-surface", "font": "--text-base",
        "appearance": "none", "placeholderColor": "--color-ink-subtle",
        "states": [
          { "state": "focus-visible", "decl": "outline: 2px solid var(--color-accent); outline-offset: 2px", "evidence": "motion-states-cssom.json#rules[22]", "verified": "pointer+Tab" },
          { "state": "invalid",  "decl": "border-color: oklch(0.58 0.19 26)", "evidence": "motion-states-cssom.json#rules[24]", "verified": "authored-not-verified" },
          { "state": "disabled", "decl": "opacity: 0.5; cursor: not-allowed",  "evidence": "motion-states-cssom.json#rules[25]", "verified": "authored-not-verified" }] },
      { "name": "plan", "type": "select", "appearance": "none", "options": ["Starter", "Team", "Enterprise"],
        "chevron": "background-image from assets[] — the native arrow is gone because appearance is none",
        "states": [{ "state": "checked", "decl": "font-weight: 600", "verified": "authored-not-verified" }] }],
    "submit": { "text": "see content.md", "type": "submit", "states": [{ "state": "disabled", "decl": "opacity: 0.4" }] },
    "instruction": "Reproduce every field's type, name, autocomplete, inputmode, required, and placeholder verbatim — they are content, not styling. Keep `appearance: none` wherever it was measured and re-author the affected affordance (select chevron, checkbox tick, range thumb). Style ::placeholder explicitly; it does not inherit color. Every state row ships, including the ones marked authored-not-verified. The form does not need to submit: no fetch, no client validation the original did not have, no library. Render it INERT — `action` omitted and submit prevented — because `actionPolicy` is `inert`: a mirrored form that keeps the original's endpoint posts a visitor's email straight to the cloned site's owner, from a domain they never agreed to. `measuredAction` is provenance only; ship it as the live `action` **only** when `actionPolicy` is `kept`, which only the user can ask for." }
}
```

Prompt deltas — append to the DoD: `[ ] Every field's type / name / autocomplete / inputmode / required /
placeholder matches spec.form.fields[] exactly.` · `[ ] appearance:none preserved and the replaced affordance
re-authored.` · `[ ] ::placeholder styled explicitly.` · `[ ] Every spec.form.*.states[] row present, including
:disabled / :invalid / :checked.` · `[ ] No client-side validation or submit handler the original did not have.` ·
`[ ] The form is inert unless spec.form.actionPolicy is "kept" — no live action, submit prevented, and the fact
recorded in report.md under Deviations.`

## 6. `requests.json`, and how the orchestrator drains it

```json
{ "requests": [
  { "kind": "token", "name": "--color-warning", "value": "oklch(0.78 0.16 78)", "why": "badge background measured on .badge--beta; nearest existing token is ΔE 9 away" },
  { "kind": "dependency", "name": "motion", "why": "scroll-linked parallax; CSS animation-timeline covers it — reject if so" },
  { "kind": "shared-component", "name": "Badge", "why": "identical markup also appears in 06 and 09" },
  { "kind": "global-css", "css": "@keyframes marquee{from{transform:none}to{transform:translateX(-50%)}}", "why": "cannot be expressed as a utility" }
] }
```

| Kind | What the agent does meanwhile |
|---|---|
| `token` | uses the nearest existing token, notes the swap in `report.md` |
| `dependency` | ships the CSS-native version; the default answer is no |
| `shared-component` | keeps its local copy; the orchestrator extracts it and rewrites both imports |
| `global-css` | approximates with utilities or omits the effect, and says which |

Drain in wave n+1, before wiring the page file:

1. `Glob .clone/sections/*/requests.json`, parse each, group by `kind` then `name`.
2. Count requesters per `(kind,name)`: two or more independent requesters for one token is an automatic accept, that being measured convergence rather than preference.
3. Apply centrally, only in files you own: `token` / `global-css` → the tokens file and base layer;
   `shared-component` → extract into the shared dir and rewrite the importing sections yourself; `dependency` → default
   no. Accept/reject rules per kind: `references/assembly.md`.
4. Record every decision in `run.json.requests.applied[]` / `rejected[]` as `{kind, name, from}`, set
   `requests.drained`, write `run.json` atomically.
5. Re-dispatch a section **only** if it still fails a gate after the change: an applied token does not by itself
   justify a repair pass, and a rejected request does not either — the agent already shipped its fallback.

## 7. `report.md` and the repair addendum

`report.md` is the five-heading shape from §3, capped at 400 words. Read reports **only** for sections that failed a
gate or flagged a deviation — reading 20 passing reports is pure token burn. A repair pass reuses the original
`PROMPT.md` and appends this addendum; same file, same export, same ownership, nothing else re-litigated:

```md
## REPAIR PASS {{N}} of {{K}} — your section failed these gates
{{FAILING_GATE_LINES}}          <!-- e.g. w1440 pixelDiff 0.071 | aHash 9 | worstTile 0.34 at x480 y320 w180 h80 -->
geometry: {{PROBE_FAIL_COUNT}} of {{PROBE_TOTAL}} probes FAIL

## Read these, in this order
1. /abs/.clone/sections/{{ID}}/geometry.md      <- exact numeric deltas, original vs yours
2. /abs/.clone/sections/{{ID}}/diff.json        <- tiles[] heat grid, row-major; worstTileBox is where to look
3. /abs/.clone/sections/{{ID}}/clone-w1440.webp <- what you actually shipped
4. /abs/.clone/sections/{{ID}}/orig-w1440.webp  <- what it must look like
5. your own report.md — you flagged {{DEVIATION_COUNT}} deviations; resolve them or explain why you cannot

## Rules
Fix ONLY the failing measurements. Do not restyle passing parts — churn regresses a good diff score.
No new dependencies. No shared-file edits. Same single file, same export name.
If a gate is unreachable without a shared change, file the request and say so in report.md. Do not fake it.
```

## 8. The scaffold to fill

Substitute every `{{…}}`; leave nothing templated. `{{GATE_TABLE}}` is pasted from `references/assembly.md`;
`{{ALLOWED_DEPS}}`, `{{TOKENS_FILE}}`, `{{EXAMPLE_FILE}}`, `{{WRITE_FILE}}`, `{{FORBIDDEN_PATHS}}` come from
`run.json.stack` (`references/stacks.md`) as literal absolute paths — they are **not** `spec.json` fields, and the
agent is forbidden to read `run.json`; `{{DOD_CHECKLIST}}` is §3's checklist plus the per-shape deltas from §4/§5.

`{{FORBIDDEN_PATHS}}` is **`sharedDir` and `sectionsDir` (and in `--static` the shared/sections dir they collapse
into) minus this agent's own `spec.writeFile` — plus `spec.styleFile` in `--static`** — followed by `tokensFile`,
the layout file, `pageFile`, every config, the lockfile, `assetsDir`/`public/**`, and `.clone/**` except this
section's own `report.md` and `requests.json`. The carve-out is not cosmetic: **wave 1 builds the shared chrome**
(`role: header|footer|sticky-cta` and every hash-deduped shared component), so those agents' `writeFile` is
`<sharedDir>/SiteHeader.tsx` and the like. A blanket "never touch the shared dir" would leave a wave-1 agent with
nowhere to write. Render it as one literal line per run, e.g. for the hero of §3:
`components/shared/** and components/sections/** except components/sections/Hero.tsx, app/globals.css,
app/layout.tsx, app/page.tsx, tailwind/postcss config, package.json, tsconfig.json, public/**`. The invariant that
actually matters is **one writer per path** — never "one directory per role". Ownership map:
`references/sectioning.md` §10.

`{{CAPTURE_LINES}}` iterates `spec.captures`, one line per measured width, never a fixed pair. At a single-width
profile (`--profile cheap`: `spec.widths == [1440]`, one capture, one viewport measured) three things change
together, or the agent is told to read a file that was never shot: the extra capture lines are absent, the DoD's
`Responsive:` row becomes `[ ] Responsive: hold the breakpoints in spec.layout.breakpointsObserved and
spec.responsive.* — only 1440 was measured, so do not invent a narrow composition; nothing overflows
horizontally at 360px`, and `spec.responsive.layout[].byViewport` / `spec.box` / `spec.layout` carry the primary
width only. Write `"profile": "cheap"` into the spec so `report.md` can say which widths it could actually judge.

In `--static` mode `{{WRITE_FILE}}` is `sections/{{ID}}.html`, `{{EXPORT_LINE}}` becomes "a single
`<section data-section="{{ID}}">` fragment, no `<html>`/`<head>`", the token source is `styles.css`, and the
per-section CSS rules go in the agent's own `{{STYLE_FILE}}` — `sections/{{ID}}.css`, its second owned file,
carried in `spec.json` as `"styleFile"` (the key is absent in every framework mode, where the utility engine
covers it). `tokenRule`, the arbitrary-value DoD row, and the write-surface row all swap to their static
wording — the table is in `references/stacks.md` §5, and that file owns it.

```md
Build ONE section component. You are one of {{WAVE_SIZE}} agents working in parallel on the same clone.
You cannot see the live site, you have no browser, and you have no shell. Your inputs are complete.

READ FIRST, in this order (absolute paths):
  1. {{ROOT}}/.clone/sections/{{ID}}/spec.json        2. {{ROOT}}/.clone/sections/{{ID}}/content.md
  3. {{CAPTURE_LINES}}     <!-- one absolute path per entry in spec.captures, widest first:
                                {{ROOT}}/.clone/sections/{{ID}}/orig-w1440.webp   <- the target at 1440
                                {{ROOT}}/.clone/sections/{{ID}}/orig-w390.webp    <- the target at 390   -->
  4. {{ROOT}}/{{TOKENS_FILE}}  <- READ ONLY. The token source.   5. {{ROOT}}/{{EXAMPLE_FILE}}  <- house style

WRITE EXACTLY ONE FILE: {{ROOT}}/{{WRITE_FILE}}
  {{EXPORT_LINE}}

YOU MAY NOT TOUCH: {{FORBIDDEN_PATHS}}
ALLOWED IMPORTS: {{ALLOWED_DEPS}} — nothing else. You cannot install packages.
NEED SOMETHING SHARED? Append to {{ROOT}}/.clone/sections/{{ID}}/requests.json
  kinds: token | dependency | shared-component | global-css. Proceed with the nearest existing token.

{{SECTION_SPECIFIC_NOTES}}        <!-- the repeat / carousel / pinned / form instruction lines from spec.json -->

DEFINITION OF DONE — self-check every line, report each verdict in report.md:
{{DOD_CHECKLIST}}

GATES you will be measured against (verbatim from references/assembly.md):
{{GATE_TABLE}}

WRITE report.md, MAX 400 WORDS: ## Built / ## Done-check / ## Deviations / ## Requests / ## Uncertainty
```

## 9. Why briefs fail

| Failure | What ships |
|---|---|
| a measured value is missing from `spec.json` | the agent invents one. If a value genuinely was not measured, write `"notMeasured"` rather than omitting the field |
| no capture, or one from the wrong width | right numbers, wrong composition — nothing catches it until the diff |
| two agents pointed at one file | last writer wins, silently, and both reports claim success |
| `content.md` summarized instead of transcribed | paraphrased copy: the text-fidelity gate fails and the verbatim-mirror claim is void |
| a remote URL left where the agent can copy it | the "clone" fetches from the origin; only `spec.assets[].ref` may appear in code |
| gate numbers retyped instead of pasted from `references/assembly.md` | it optimizes toward the wrong target and passes its own self-check |
| a `motion` / `responsive` entry with no `evidence` | invented animation, defended in `report.md` as measured |
