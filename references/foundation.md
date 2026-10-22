# Foundation — turning `foundation.json` into the clone's code

Owns the `foundation`→code step: token emission, `@font-face` against the **mirrored local files**, the base
layer, dark theme, the layout shell, `_EXAMPLE`. Inputs: `.clone/foundation.json`, `.clone/assets.json` (local
font paths), `.clone/responsive.json` (breakpoint boundaries, solved fluid values). Where files go is
`references/stacks.md` + `run.json.stack` — never hardcode a path here; how the numbers were measured is
`references/extraction.md`; font *file* download is `references/assets.md`; section components are
`references/agent-brief.md`; gate numbers are `references/assembly.md`.

Two modes, one vocabulary. Tailwind v4 `@theme` namespaces are load-bearing (they generate the utilities), so
`--static` reuses the **identical names** inside `:root` and only loses the generated classes.

## 1. Palette → semantic roles

Three inputs, in order. (1) `color.vars[]` — the page's own custom-property names are the best evidence of
intent; map them onto the role vocabulary, keep the values verbatim. (2) `color.palette[]` — clustered and
weight-ranked by painted area (surfaces), ink area (text) or perimeter×width (lines), so index 0 is the dominant
surface on essentially every page. (3) `color.byRole` — unclustered per-role rankings for tie-breaking: a value
in `byRole.text` is ink, in `byRole.background` a surface, in both usually an inverse pair. The role vocabulary
below is closed: do not invent names, add numeric variants only when measured.

| Role token | Assign from |
|---|---|
| `--color-surface` | highest-weight `palette[]` entry whose `roles[]` contains `background` |
| `--color-surface-raised` | next background cluster whose OKLab L differs by ≥ 0.02 *away* from ink (cards, panels) |
| `--color-surface-sunken` | background cluster on the other side of `surface` (wells, code blocks, alternating bands) |
| `--color-surface-inverse` | background cluster whose L is on the opposite side of 0.5 from `surface` |
| `--color-ink` | highest-weight entry whose `roles[]` contains `text` |
| `--color-ink-muted` / `--color-ink-subtle` | next two `text` entries, each ΔL ≥ 0.06 further toward `surface` |
| `--color-ink-inverse` | the `text` color actually measured on `surface-inverse` — check `fingerprint[].bg` |
| `--color-border-subtle` / `--color-border-strong` | the two heaviest `roles[]∋border` entries, split by OKLab distance from `surface` |
| `--color-accent` | highest-**chroma** entry (`√(a²+b²)` of `palette[].oklab`) appearing on a `button`, `a` or `[class*=cta]` row of `fingerprint[]`. Not the highest weight — brand color is small and loud |
| `--color-accent-soft` / `--color-accent-strong` | same hue (Δhue ≤ 12°), lower / higher chroma or L |
| `--color-accent-contrast` | the `text` color measured on the accent surface |
| `--color-brand` | keep the site's own name too when it clearly has one brand color; an alias, not a replacement for `accent` |
| `--color-success` / `warning` / `danger` / `info` | only when a green / amber / red / blue cluster exists with a matching class or role in `fingerprint[]` |

Non-negotiable: emit the **measured string** — `oklch(0.62 0.19 258)` stays as written, because `palette[].rgba`
/`oklab` are 8-bit clustering artefacts. Collapse each cluster to **one** token (`members[]` lists the
near-duplicates it absorbed: license to delete four near-blacks, not to ship four tokens). Alpha-bearing values
keep their alpha (`rgb(15 23 42 / 0.08)`) instead of becoming an opacity utility. Four to eight surface/ink
tokens plus two to four accent tokens is a real page; twelve grays means you skipped clustering.

```css
/* Tailwind v4 — @theme must be at the top level of the entry stylesheet. No config file. */
@import "tailwindcss";
@theme {
  /* --color-*: initial;  <- uncomment to drop Tailwind's 22 default palettes entirely */
  --color-surface: oklch(0.99 0.002 250); --color-surface-raised: #ffffff;
  --color-surface-sunken: oklch(0.965 0.004 250); --color-surface-inverse: oklch(0.21 0.02 258);
  --color-ink: rgb(15, 23, 42); --color-ink-muted: rgb(71, 85, 105); --color-ink-subtle: rgb(148, 163, 184);
  --color-ink-inverse: oklch(0.98 0.003 250);
  --color-border-subtle: rgb(226, 232, 240); --color-border-strong: rgb(203, 213, 225);
  --color-accent: oklch(0.62 0.19 258); --color-accent-soft: oklch(0.95 0.03 258);
  --color-accent-strong: oklch(0.52 0.2 258); --color-accent-contrast: #ffffff;
}
```

`--color-accent` generates `bg-accent`, `text-accent`, `border-accent`, `ring-accent`, `from-accent`, … In
`--static` the same names go in `:root` and sections consume them directly; inline `style` attributes are
forbidden, write real rules in `styles.css`:

```css
:root { color-scheme: light dark; --color-surface: oklch(0.99 0.002 250); --color-ink: rgb(15, 23, 42);
        --color-accent: oklch(0.62 0.19 258); /* …one line per token, same names as above… */ }
.hero { background: var(--color-surface); color: var(--color-ink); }
```

### The step-500 trap

When the measured brand has a real ramp — `members[]` spread across several L values, or `color.vars[]` holding
`--blue-100 … --blue-900` — it is tempting to spread the measured swatches over evenly spaced step numbers. Do not:

```css
/* WRONG — six measured swatches mapped onto six evenly spaced indices */
@theme {
  --color-blue-100: oklch(0.95 0.03 258);  --color-blue-300: oklch(0.80 0.10 258);
  --color-blue-400: oklch(0.71 0.15 258);  /* 500 skipped */
  --color-blue-600: oklch(0.52 0.19 258);  --color-blue-800: oklch(0.38 0.14 258);
}
```

`bg-blue-500` still compiles, because **Tailwind v4's own default `--color-blue-500` answers it**: a section that
reaches for the mid step ships stock Tailwind blue beside your measured brand — no build error, no lint warning,
a pixel delta small enough to survive a glance. Same hole for every default hue name (`slate`, `gray`, `zinc`,
`red`, `green`, `blue`, `indigo`, …). Fix, both halves:

```css
@theme {
  --color-blue-*: initial;                  /* 1. clear the inherited ramp for this name */
  --color-blue-50:  oklch(0.98 0.012 258); --color-blue-100: oklch(0.95 0.03 258);
  --color-blue-200: oklch(0.90 0.06 258);  --color-blue-300: oklch(0.83 0.10 258);
  --color-blue-400: oklch(0.72 0.15 258);
  --color-blue-500: oklch(0.62 0.19 258);   /* 2. the measured brand value, anchored on 500 */
  --color-blue-600: oklch(0.55 0.19 258);  --color-blue-700: oklch(0.48 0.17 258);
  --color-blue-800: oklch(0.40 0.14 258);  --color-blue-900: oklch(0.33 0.10 258);
  --color-blue-950: oklch(0.24 0.07 258);
}
```

Anchor **500 on the measured brand value**, fill *every* standard step (`50 100 200 300 400 500 600 700 800 900
950`) by interpolating L between measured neighbours at constant hue, and list the interpolated steps in
`CLONE-REPORT.md` as derived. Cheaper alternative when the page only ever used three swatches: name the ramp
something Tailwind does not ship (`--color-brand-*`) and emit only the measured steps — nothing can fall
through. Either way no utility may resolve to a default:

```bash
grep -ohrE '(bg|text|border|ring|from|via|to)-[a-z]+-[0-9]{2,3}' components/ app/ | sort -u   # steps in use
grep -c -- '--color-blue-500' "$(jq -r .stack.tokensFile .clone/run.json)"                    # must be >= 1
```

## 2. Type

Map `type.scale[]` ascending onto `--text-*` rungs; convert px→rem at `meta.rootFontSizePx` (usually 16 — do
not assume). `type.roles[]` is char-weight-ranked, so its top rows are the page's real type styles.

| Measured | Token |
|---|---|
| `type.scale[].sizeRem` | `--text-xs` … `--text-7xl`, ascending |
| `type.roles[].lhRatio` for that size | `--text-<rung>--line-height`, **unitless ratio** |
| `type.roles[].trackingEm` | `--text-<rung>--letter-spacing`, in `em` |
| dominant `fontWeight` for that size | `--text-<rung>--font-weight` |
| `fonts.usage[]` by `charWeight` | `--font-sans` (rank 1), `--font-display` (the `h1`/`h2` family if different), `--font-mono` (the `code`/`pre` family) |
| `lhNormal === true` | emit `line-height: normal`, **not** the measured px — it drifts with the font file |
| `trackingNormal === true` | omit `letter-spacing` entirely |
| `type.fluid[]` with `prop:"font-size"`, `kind:"clamp"` | the rung's value becomes the solved `cssRem` |
| that entry's `minIsObservedOnly` / `maxIsObservedOnly` | emit it anyway, and record in `CLONE-REPORT.md` that the rail is observed, not authored — the fluid band between the rails is always right |

```css
@theme {
  --font-sans: "Söhne", ui-sans-serif, system-ui, -apple-system, sans-serif;
  --font-display: "Söhne Breit", var(--font-sans);   --font-mono: "Söhne Mono", ui-monospace, Menlo, monospace;
  --text-base: 1rem;   --text-base--line-height: 1.6;
  --text-2xl: 1.5rem;  --text-2xl--line-height: 1.25;  --text-2xl--letter-spacing: -0.01em;
  --text-6xl: clamp(2.25rem, 1.2rem + 4.2vw, 4.5rem);          /* solved from the 7-width sweep */
  --text-6xl--line-height: 1.05; --text-6xl--letter-spacing: -0.03em; --text-6xl--font-weight: 600;
}
```

A `clamp()` inside `--text-*` is legal — the value lands verbatim in `font-size`. In `--static` the same three
properties become a class: `.t-6xl { font-size: clamp(2.25rem, 1.2rem + 4.2vw, 4.5rem); line-height: 1.05;
letter-spacing: -0.03em; font-weight: 600; }`

### `@font-face` — local files only

One block per `fonts.faces[]` entry whose `status` is `loaded` or which appears in the network census. `src`
points at the mirrored bytes — `assets.json` → **`ref`**, the public path (`/assets/fonts/…`), never `local` (which
is staging-relative, `fonts/soehne.woff2`, and would 404 from every nested route) and never a CDN: a clone that
fetches `use.typekit.net`
is a hotlink, and it breaks offline and on the first CORS change.

```css
/* not inside @theme — @font-face is a declaration, not a token */
@font-face {
  font-family: "Söhne";
  src: url("/assets/fonts/soehne-var-latin.woff2") format("woff2-variations"),
       url("/assets/fonts/soehne-var-latin.woff2") format("woff2");
  font-weight: 400 700;   /* faces[].weight verbatim — a range means variable */
  font-style: normal;
  font-display: swap;     /* faces[].display verbatim; only default to swap when it was `auto` */
  unicode-range: U+0000-00FF, U+0131, U+0152-0153;   /* faces[].unicodeRange verbatim */
  size-adjust: 104%;      /* faces[].sizeAdjust / ascentOverride / descentOverride — omit when null */
}
```

- **One `@font-face` per `unicode-range` subset.** Collapsing latin + latin-ext + cyrillic into one rule makes
  the browser download every subset on every page and changes which file renders which glyph.
- Variable faces get the range weight and the `woff2-variations` hint first; never one face per weight. Static
  faces get one block per weight. When `fonts.synthesisRisk[]` names a weight you have a choice, and it goes in
  `CLONE-REPORT.md`: ship the real file if the census has it, or reproduce the synthesis
  (`font-synthesis: weight`) so the clone matches what the original actually renders.
- **Declared weight ≠ rendered weight.** Builders routinely declare the same family twice — once fully specified,
  once with no `font-weight` descriptor, which defaults to `400` — so the face whose descriptor you copied may not
  be the one that rendered. Trust the measured **rendered** value (`type.roles[].fontWeight`,
  `fingerprint[].fontWeight`) and emit faces that cover it; `faces[].weight` only tells you what was declared.
  `fingerprint[].textWidthPx` is the gate that catches getting this backwards.
- Axis inventories are not in the DOM — read `fvar` offline from the mirrored file if a section needs an axis
  beyond `wght`; otherwise pass `type.roles[].variationSettings` verbatim (`font-variation-settings: "opsz" 32`).
- `next/font/local` pointing at the same mirrored files adds preload and an adjusted fallback metric. Use it
  only when the project already uses `next/font`; plain `@font-face` keeps both modes identical.

## 3. Spacing

Tailwind v4 derives its whole dynamic spacing scale from one value, so in the common case you emit one token
and `p-4`, `gap-6`, `mt-10`, `size-3.5` all follow: `@theme { --spacing: 4px; }` — that is `space.baseUnitPx`,
valid when `baseUnitConfidence ≥ 0.85`. Add `--space-*` aliases **only** when `space.scale[]` is not a set of
multiples of `baseUnitPx` (a 4-8-12-16-24-40-64-104 ladder):

```css
@theme { --spacing: 4px;
  --space-2xs: 0.25rem; --space-xs: 0.5rem; --space-sm: 0.75rem; --space-md: 1rem;
  --space-lg: 1.5rem;   --space-xl: 2.5rem; --space-2xl: 4rem;   --space-3xl: 6.5rem; }
```

When `baseUnitConfidence < 0.85` the page has no grid: emit the observed `space.scale[]` as `--space-*`, omit
`--spacing`, and say so in `CLONE-REPORT.md`. Do not round values to manufacture a grid nobody authored.
`--spacing` alone does nothing in `--static`, so static mode always emits the `--space-*` ladder.

## 4. Radius, shadow, border, motion tokens

Take the top of each weight-ranked `patterns` list. Two to four rungs each; a fifth is noise.

| Source | Token | Note |
|---|---|---|
| `patterns.radii[]` | `--radius-sm/md/lg/xl`, `--radius-full` | `9999px` and `50%` are different intents — keep what was measured, do not normalize |
| `patterns.shadows[]` | `--shadow-xs/sm/md/lg/xl` | verbatim, including Chrome's color-first serialization; a multi-layer shadow stays one token |
| `patterns.borders[]` | width + style + a color token, inline | Tailwind v4 has no `--border-*` namespace |
| `patterns.easings[]` | `--ease-out-quart`, `--ease-spring` | copy `cubic-bezier()`/`linear()` **verbatim** — a `linear()` point list is a spring approximation and rounding it kills the feel |
| `patterns.durations[]` | `--clone-dur-fast/base/slow` | no `--duration-*` theme namespace exists; `--clone-*` is the non-utility prefix |
| `patterns.animations[]` + `keyframes[]` | `--animate-<name>` + the `@keyframes` block | keyframes verbatim; rebuild semantics are `references/motion.md` |

```css
@theme { --radius-sm: 4px; --radius-md: 8px; --radius-lg: 14px; --radius-full: 9999px;
  --shadow-sm: 0 1px 2px 0 rgb(15 23 42 / 0.06);
  --shadow-lg: 0 12px 32px -8px rgb(15 23 42 / 0.18), 0 2px 6px -2px rgb(15 23 42 / 0.08);
  --ease-out-quart: cubic-bezier(0.165, 0.84, 0.44, 1); --animate-marquee: marquee 28s linear infinite; }
@keyframes marquee { from { transform: translateX(0); } to { transform: translateX(-50%); } }
```

Reproduce every `color.registered[]` entry as an `@property` rule before anything transitions it, or the
transition silently does nothing: `@property --clone-scroll-progress { syntax: "<number>"; inherits: false;
initial-value: 0; }`

## 5. Layout shell

Breakpoints come from `responsive.json.breakpointSystem.boundaries`. In Tailwind v4, `--breakpoint-*` in
`@theme` **is** the variant generator — `--breakpoint-md: 48rem` creates the `md:` variant. Use `rem` so the
value tracks the same 16px divisor the original media queries used.

```css
@theme { --breakpoint-sm: 40rem; --breakpoint-md: 48rem; --breakpoint-lg: 64rem;
  --breakpoint-xl: 80rem; --breakpoint-2xl: 96rem;
  --container-prose: 42.5rem;   /* authoredMaxWidths: 68ch measured 680px */
  --container-6xl: 72rem; }     /* the dominant contentWidths mode */
```

There is **no** `@custom-media` in Tailwind v4 or in browsers — it is a PostCSS plugin, and
`references/stacks.md` allows no extra deps. So Tailwind mode gets variants from `--breakpoint-*`; `--static`
writes plain `@media (min-width: 48rem)` and still declares `--breakpoint-md` in `:root` for documentation only
(custom properties are not valid inside a media condition).

The base layer holds what is a *document* fact rather than a token. Emit only lines you measured — an unmeasured
`scroll-behavior: smooth` is a design decision, not a clone.

```css
@layer base {
  html { scroll-behavior: smooth;              /* topology.scroll.behavior */
         scroll-padding-top: 4.5rem;           /* topology.scroll.scrollPaddingTop, or the sticky header height */
         -webkit-text-size-adjust: 100%; }
  body { background: var(--color-surface); color: var(--color-ink); font-family: var(--font-sans);
         font-size: var(--text-base); line-height: var(--text-base--line-height);
         font-feature-settings: "ss01", "cv11";   /* fonts.featureSettings[0], verbatim */
         overscroll-behavior: none; }            /* topology.scroll.overscroll */
  ::selection { background: var(--color-accent-soft); color: var(--color-ink); }
  :focus-visible { outline: 2px solid var(--color-accent); outline-offset: 2px; }
}
```

The shell is three things: the tokens/base stylesheet (`tokensFile`); the page file (`pageFile`) mounting the
shared header, the ordered section list and the shared footer, carrying `meta.title`, `meta.lang`, `meta.dir`,
**the viewport meta** and `assets.ogImage`; and the header/footer components in `sharedDir`. `--static`'s
`index.html` gets the same `<html lang dir>`, the same viewport meta, plus
`<link rel="stylesheet" href="styles.css">`, each section inlined from `sections/<id>.html`.

**The viewport meta is measured, and it is the one document element whose absence no gate can see.** Emit
`responsive.json.metaViewport` verbatim (`references/responsive.md` R9 records it from
`breakpoints().metaViewport`); when the original had none, emit the modern default:

```html
<meta name="viewport" content="width=device-width, initial-scale=1" />
```

Without it a phone lays the page out in the ~980px legacy viewport and every narrow style is dead — and **the
clone still passes**, because every narrow capture and every overflow probe runs under `cdp:resize_page` /
`cdp:emulate`, which override device metrics and never read the tag. In Next App Router the same value goes in
the `viewport` export (`export const viewport = { width: "device-width", initialScale: 1 }`), not a hand-written
`<meta>` in `<head>`; if the original carried extra directives (`maximum-scale`, `user-scalable=no`,
`viewport-fit=cover`) reproduce them verbatim and note `user-scalable=no` in `CLONE-REPORT.md` as an
accessibility deviation the original owns.

Reproduce `topology.zLadder[]` as a semantic ladder — `--clone-z-sticky`, `--clone-z-overlay`, `--clone-z-toast`,
mapped from the measured `z` values ascending — never raw `999`s. Every `topology.stackingContexts[]` entry must
survive: dropping `contain: paint` or `isolation: isolate` changes paint order, and that bug is invisible in a
pixel diff until something clips.

## 6. Dark mode

Only when `color.darkMode` shows signal. `@theme` variables are static — `@media` cannot go inside `@theme` —
so dark values override the same custom properties under a selector or media query, in `@layer base`.

```css
/* hasMediaQuery: true, no toggleSelectors */
@layer base {
  @media (prefers-color-scheme: dark) {
    :root { --color-surface: oklch(0.17 0.015 258); --color-surface-raised: oklch(0.22 0.017 258);
            --color-ink: oklch(0.97 0.004 250); --color-ink-muted: oklch(0.72 0.012 255);
            --color-border-subtle: oklch(0.3 0.015 258); --color-accent: oklch(0.72 0.16 258); } } }

/* toggleSelectors non-empty: register the variant so dark: utilities work, then override under the
   same selector the original used */
@custom-variant dark (&:where(.dark, .dark *));
@layer base { .dark { --color-surface: oklch(0.17 0.015 258); --color-ink: oklch(0.97 0.004 250); } }
```

Every override comes from `color.darkMode.tokenDiff[]` (`{name, light, dark}`). A token absent from the diff
must **not** be overridden — the original kept it, so the clone keeps it. Set `color-scheme: light dark` on
`:root` when `meta.colorSchemeUsed` says so: it themes scrollbars and form controls, invisibly to a screenshot.

## 7. `_EXAMPLE`

One section component that no page imports, in `sectionsDir`, as the house style every `clone-section` agent
copies. It is a contract, not decoration. In ~40 lines it must demonstrate: named export, no default; a
`data-section` attribute; the container + full-bleed pattern; token-only colors and spacing (**zero** literal
hex, rgb or px); an `aria-labelledby` heading link; one responsive change as a breakpoint variant (or a plain
`@media` block in `--static`); one `<img>` with explicit `width`/`height` pointing at a mirrored asset; and a
`prefers-reduced-motion` guard on its single transition. No client JS unless the section needs it.

## 8. Exit gate

The foundation is done when all of these hold. Then write `run.json.foundation` (`tokensFile`, `tokenCount`,
`fontFamilies[]`, `hash`, `breakpoints[]`) atomically and fan out.

- [ ] `tokensFile` compiles — the stack's build/typecheck is clean, and in Tailwind mode a throwaway
      `bg-surface text-ink p-4` resolves to real CSS instead of being dropped as unknown.
- [ ] Every `fonts.usage[].isWebfont` family has an `@font-face` whose `src` resolves to a file that exists
      under `assetsDir` and is listed in `.clone/assets.json` with a sha256. **Zero** remote font URLs remain:
      `grep -rE "https?://[^\"')]+\.(woff2?|ttf|otf)" <tokensFile>` returns nothing.
- [ ] Every `unicode-range` subset is its own `@font-face`; every variable face uses a range weight.
- [ ] Token counts are sane: 6–16 `--color-*`, 4–10 `--text-*`, 2–5 `--radius-*`, 2–5 `--shadow-*`, one
      `--spacing` **or** a full `--space-*` ladder, one `--breakpoint-*` per
      `responsive.json.breakpointSystem.boundaries` entry.
- [ ] `run.json.foundation.tokenCount > 0`, and **zero** requests to `fonts.googleapis.com`, `fonts.gstatic.com`,
      `use.typekit.net`: `grep -rInE 'fonts\.(googleapis|gstatic)\.com|use\.typekit\.net' <tokensFile> app components public`
      returns nothing.
- [ ] Every mirrored file has a `.clone/PROVENANCE.md` row with source URL, content-type, bytes and sha256
      (`references/assets.md`).
- [ ] No `bg-<hue>-<step>` anywhere in the tree resolves to a Tailwind default ramp (§1, the step-500 trap).
- [ ] Every `color.registered[]` entry has an `@property` rule.
- [ ] Every `color.darkMode.tokenDiff[]` name is overridden in the dark block, and nothing else is.
- [ ] The shell renders: header, empty ordered section list, footer, correct `<html lang dir>`, title from
      `meta.title`, no console errors.
- [ ] The viewport meta is present and equals `responsive.json.metaViewport` verbatim, or
      `width=device-width, initial-scale=1` when the original had none:
      `grep -rn 'width=device-width' <pageFile> app/layout.* index.html` returns a hit.
- [ ] `_EXAMPLE` exists, contains no literal color or spacing value, and is imported by nothing.

## 9. Worked example

`.clone/foundation.json`, elided:

```jsonc
{ "meta": { "rootFontSizePx":16, "colorSchemeUsed":"light dark", "lang":"en" },
  "fonts": { "faces":[{ "family":"Söhne","weight":"400 700","style":"normal","display":"swap","status":"loaded",
               "unicodeRange":"U+0000-00FF, U+0131","sizeAdjust":null,"isVariable":true,"source":"both",
               "srcUrls":["https://cdn.acme.com/f/soehne-var.woff2"] }],
             "usage":[{"resolvedFamily":"Söhne","isWebfont":true,"charWeight":18422}],
             "featureSettings":["\"ss01\""], "synthesisRisk":[] },
  "color": { "palette":[{"value":"rgb(255, 255, 255)","weight":2914000,"roles":["background"],"members":["#fff"]},
               {"value":"rgb(15, 23, 42)","weight":41200,"roles":["text","background"],"members":["#0f172a"]},
               {"value":"oklch(0.62 0.19 258)","oklab":[0.62,-0.03,-0.18],"weight":12700,"roles":["background","text"]},
               {"value":"rgb(226, 232, 240)","weight":9100,"roles":["border"]},
               {"value":"rgb(100, 116, 139)","weight":7300,"roles":["text"]}],
             "registered":[{"name":"--brand-shift","syntax":"<angle>","inherits":false,"initialValue":"0deg"}],
             "darkMode":{ "hasMediaQuery":true,"toggleSelectors":[],
               "tokenDiff":[{"name":"--bg","light":"rgb(255, 255, 255)","dark":"oklch(0.17 0.015 258)"},
                            {"name":"--fg","light":"rgb(15, 23, 42)","dark":"oklch(0.97 0.004 250)"}] } },
  "type": { "scale":[{"sizePx":14,"sizeRem":0.875},{"sizePx":16,"sizeRem":1},{"sizePx":24,"sizeRem":1.5},{"sizePx":72,"sizeRem":4.5}],
            "roles":[{"role":"p","fontSizePx":16,"lhRatio":1.6,"trackingNormal":true,"fontWeight":"400"},
                     {"role":"h1","fontSizePx":72,"lhRatio":1.05,"trackingEm":-0.03,"fontWeight":"600"}],
            "fluid":[{"path":"h1","prop":"font-size","kind":"clamp","cssRem":"clamp(2.25rem, 1.2rem + 4.2vw, 4.5rem)"}] },
  "space": { "baseUnitPx":4,"baseUnitConfidence":0.97,
             "containers":{"contentWidths":[{"value":1152}],"authoredMaxWidths":[{"value":"68ch","selector":"article p"}]} },
  "patterns": { "radii":[{"value":"8px","weight":88000},{"value":"9999px","weight":21000}],
                "shadows":[{"value":"rgba(15, 23, 42, 0.06) 0px 1px 2px 0px","weight":64000}],
                "easings":["cubic-bezier(0.165, 0.84, 0.44, 1)"], "breakpoints":[{"px":768,"count":22}] },
  "topology": { "scroll":{"behavior":"smooth","overscroll":"none","scrollPaddingTop":"72px"},"zLadder":[{"z":10},{"z":40},{"z":60}] } }
```

The `globals.css` it produces, complete:

```css
@import "tailwindcss";

@font-face { font-family: "Söhne";
  src: url("/assets/fonts/soehne-var.woff2") format("woff2-variations"),
       url("/assets/fonts/soehne-var.woff2") format("woff2");
  font-weight: 400 700; font-style: normal; font-display: swap; unicode-range: U+0000-00FF, U+0131; }
@property --brand-shift { syntax: "<angle>"; inherits: false; initial-value: 0deg; }

@theme {
  --font-sans: "Söhne", ui-sans-serif, system-ui, sans-serif;
  --color-surface: rgb(255, 255, 255);        --color-surface-sunken: rgb(248, 250, 252);
  --color-surface-inverse: rgb(15, 23, 42);   --color-ink: rgb(15, 23, 42);
  --color-ink-muted: rgb(100, 116, 139);      --color-ink-inverse: rgb(255, 255, 255);
  --color-border-subtle: rgb(226, 232, 240);  --color-accent: oklch(0.62 0.19 258);
  --color-accent-contrast: rgb(255, 255, 255);
  --text-sm: 0.875rem;  --text-sm--line-height: 1.5;
  --text-base: 1rem;    --text-base--line-height: 1.6;
  --text-2xl: 1.5rem;   --text-2xl--line-height: 1.25;
  --text-6xl: clamp(2.25rem, 1.2rem + 4.2vw, 4.5rem);
  --text-6xl--line-height: 1.05; --text-6xl--letter-spacing: -0.03em; --text-6xl--font-weight: 600;
  --spacing: 4px;  --radius-md: 8px;  --radius-full: 9999px;
  --shadow-sm: rgba(15, 23, 42, 0.06) 0px 1px 2px 0px;
  --ease-out-quart: cubic-bezier(0.165, 0.84, 0.44, 1);
  --breakpoint-sm: 40rem; --breakpoint-md: 48rem; --breakpoint-lg: 64rem; --breakpoint-xl: 80rem;
  --container-prose: 42.5rem;  --container-6xl: 72rem;
}

@layer base {
  :root { color-scheme: light dark; --clone-z-sticky: 10; --clone-z-overlay: 40; --clone-z-toast: 60; }
  html { scroll-behavior: smooth; scroll-padding-top: 72px; -webkit-text-size-adjust: 100%; }
  body { background: var(--color-surface); color: var(--color-ink); font-family: var(--font-sans);
         font-size: var(--text-base); line-height: var(--text-base--line-height);
         font-feature-settings: "ss01"; overscroll-behavior: none; }
  :focus-visible { outline: 2px solid var(--color-accent); outline-offset: 2px; }
  @media (prefers-color-scheme: dark) {
    :root { --color-surface: oklch(0.17 0.015 258); --color-ink: oklch(0.97 0.004 250); }
  }
}
```

Every value traces to a field above. Note what did **not** happen: `rgb(226, 232, 240)` became one
`--color-border-subtle` rather than a `slate-200`; `9999px` was kept as `--radius-full` and not normalized to
`50%`; `p`'s `trackingNormal` meant no `--text-base--letter-spacing` line at all. The one derived number is
`--container-prose: 42.5rem` — `68ch` measured 680px on the article's own font, 680 ÷ 16 = 42.5. A derived
token is a deviation, and deviations get listed in `CLONE-REPORT.md`.
