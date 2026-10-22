# Motion capture and rebuild

Read this **before you navigate**: the hook script only works if installed on the navigation that loads the
page. Output: `.clone/motion.json` (schema `clone-site/motion@1`) plus raw dumps in `.clone/raw/motion-*.json`.
`cdp:` = `mcp__plugin_chrome-devtools-mcp_chrome-devtools__`.

A screenshot is one cell of a cross product — viewport × scroll position × interaction state × moment in time.
Motion lives in the other cells. Four routes reach them; none is sufficient alone.

| Route | Sees | Blind to |
|---|---|---|
| CSSOM walk | authored `@keyframes`, `@property`, transition rules, `:hover` text, reduced-motion variants, scroll/view timelines | anything a library computes at runtime; resolved px values |
| `document.getAnimations()` | every animation the browser is *actually* running, resolved, with real keyframes and timing | rAF-driven libraries (GSAP default path, anime v3, Locomotive); animations that have not started or were GC'd |
| `initScript` hooks | WAAPI calls, `IntersectionObserver` construction, every `animation*`/`transition*` event that already fired | nothing library-internal that avoids all three |
| real pointer (`cdp:hover`, `cdp:press_key`) | resolved `:hover` / `:focus-visible` values, mid-flight `CSSTransition` keyframes | `:active`, `:disabled`, `:checked`, anything not currently visible |

## M0 — the hooks, installed before the page's own scripts

`initScript` takes a **script**, not a function, so this is a bare IIFE, passed as
`cdp:navigate_page({url, initScript: <this block>})`. It applies to the **next navigation only** — re-pass it
on every `cdp:navigate_page`, including `type:"reload"`.

```js
(() => {
  const L = (window.__cloneHooks = { animate: [], events: [], observers: [], rafBursts: 0, errors: [] });
  const safe = (v) => { try { return JSON.parse(JSON.stringify(v)); } catch (e) { return String(v); } };
  const site = () => (new Error().stack || '').split('\n').slice(2, 5).map((s) => s.trim()).join(' | ');

  const origAnimate = Element.prototype.animate;
  Element.prototype.animate = function (kf, opts) {
    if (L.animate.length < 800) L.animate.push({ t: +performance.now().toFixed(1), tag: this.tagName,
      id: this.id || null, cls: String(this.className || '').slice(0, 140),
      keyframes: safe(kf), options: safe(opts), stack: site() });
    return origAnimate.call(this, kf, opts);
  };

  const OIO = window.IntersectionObserver;
  if (OIO) {
    const Patched = function (cb, opts) {
      if (L.observers.length < 300) L.observers.push({ t: +performance.now().toFixed(1),
        rootIsViewport: !opts || !opts.root, rootMargin: (opts && opts.rootMargin) || '0px',
        threshold: opts && opts.threshold, stack: site() });
      return new OIO(cb, opts);
    };
    Patched.prototype = OIO.prototype;
    window.IntersectionObserver = Patched;
  }

  for (const type of ['animationstart', 'animationend', 'animationiteration',
                      'transitionrun', 'transitionstart', 'transitionend', 'transitioncancel']) {
    document.addEventListener(type, (e) => {
      if (L.events.length > 6000) return;
      const el = e.target;
      L.events.push({ type, t: +performance.now().toFixed(1), tag: (el && el.tagName) || null,
        cls: String((el && el.className) || '').slice(0, 100),
        name: e.animationName || e.propertyName || null, elapsed: e.elapsedTime, pseudo: e.pseudoElement || '' });
    }, true);
  }

  const origRaf = window.requestAnimationFrame;
  let n = 0, mark = 0;
  window.requestAnimationFrame = function (cb) { n++; return origRaf.call(window, cb); };
  setInterval(() => { if (n - mark > 20) L.rafBursts++; mark = n; }, 500);

  window.addEventListener('error', (e) => { if (L.errors.length < 20) L.errors.push(String(e.message)); });
})()
```

`rafBursts` counts 500 ms windows with more than 20 rAF callbacks — the fingerprint of a rAF-driven library.
Non-zero `rafBursts` with an empty `animate[]` means GSAP/anime/Locomotive: plan on §M6 empirical fitting.
Stated limit: the `IntersectionObserver` wrapper breaks pages that subclass it. If
`cdp:list_console_messages` shows new errors right after this navigation, re-navigate with hooks 1 and 3 only
and record the loss in `motion.json.warnings`.

## M1 — the nine passes

Read `scripts/extract-motion.js` with `Read` and pass its **entire contents** as the `function` argument to
`cdp:evaluate_script`, always with a `filePath` of `.clone/raw/<pass>.json`. That call installs
`window.__clone.motion` and returns `{ok,installed,utilFilled,hooks}`; every later pass is a one-line
`function` argument. Re-install after any `cdp:navigate_page`; `cdp:resize_page` does not wipe it.
`utilFilled` lists the `window.__clone.util` helpers this payload added on top of whatever
`scripts/extract-foundation.js` installed — install order does not matter, helpers are captured at install
time.

| # | Pass | `function` argument | `filePath` |
|---|---|---|---|
| 1 | libraries | `() => window.__clone.motion.libs()` | `.clone/raw/motion-libs.json` |
| 2 | CSSOM motion index | `() => window.__clone.motion.cssom()` | `.clone/raw/motion-cssom.json` |
| 3 | resting transitions | `() => window.__clone.motion.transitions()` | `.clone/raw/motion-transitions.json` |
| 4 | running animations | `() => window.__clone.motion.anims('load')` | `.clone/raw/motion-anims-load.json` |
| 5 | scroll sweep + fit | `() => window.__clone.motion.scroll(21)` | `.clone/raw/motion-scroll.json` |
| 6 | drain the hook log | `() => window.__clone.motion.drain()` | `.clone/raw/motion-hooks.json` |
| 7 | states, CSSOM route | `() => window.__clone.motion.states()` | `.clone/raw/motion-states-cssom.json` |
| 8 | states, pointer route | `() => window.__clone.motion.states('<sel>', 220)` | `.clone/raw/motion-states-live.json` |
| 9 | page scroll chrome | `() => window.__clone.motion.page()` | `.clone/raw/motion-page.json` |
| 10 | declarative motion vocabulary | install `scripts/extract-motion-attrs.js`, then `() => window.__clone.motionAttrs.vocabulary()` | `.clone/raw/motion-vocab.json` |
| 11 | motion attributes, page-wide | `() => window.__clone.motionAttrs.all()` | `.clone/raw/motion-attrs.json` |

Order matters more than the numbering suggests: four later operations destroy exactly what these passes read.

| Rule | Why |
|---|---|
| Passes 1–4 and 6 run **at load, before the prewarm tile-scroll** | the prewarm exists to fire every `IntersectionObserver` reveal and run every entrance animation to completion. After it, `anims('load')` returns the post-reveal set (mostly empty) and the hook log's pre-reveal evidence — `rootMargin`, `threshold`, the WAAPI keyframes M10's reveal rebuild is built from — is already spent. This is the one ordering mistake that cannot be repaired later at any price. |
| Pass 4 **before** pass 5 | the sweep destroys the at-load animation set. |
| Pass 5 **before** the responsive sweep (`references/responsive.md`) | a mid-page scroll position changes which reveals have fired, and therefore every computed opacity measured afterwards. |
| Every pass **before** the fluid sweep and before `cdp:emulate {colorScheme:"dark"}` | seven resizes re-run every media query, every `ResizeObserver`, and every width-conditional animation; `emulate` can trigger a reload, which wipes `window.__cloneHooks` outright (`references/extraction.md` §5). |

Passes 5, 7, 8, 9 are the post-prewarm half: they need a scrolled, revealed, fully-decoded page.
Passes 10–11 also run post-prewarm, so lazy sections exist. They are cheap and they are the only
passes that survive a fully minified bundle, so run them even when §M5c cannot.

**After any reload, hooks are gone.** `initScript` applies to one navigation, so an explicit `type:"reload"`, an
`emulate`-triggered reload, a consent-banner redirect, or an SPA route change all leave you instrumentless. Detect
it for one call's cost — `() => ({ v: window.__clone && window.__clone.v, hooks: !!window.__cloneHooks })`; a null
`v` means every payload is gone too. Recover in this order: `cdp:navigate_page {url, type:"url", initScript: HOOKS}`
→ re-install `extract-sections.js`, `extract-motion.js`, `probe-responsive.js`, `extract-foundation.js` → re-run
motion passes 1–4 and 6 → re-prewarm. Anything already drained to `.clone/raw/motion-hooks.json` survives; anything
not drained is unrecoverable, so write `motion.json.warnings: ["hooks lost to reload before drain — reveal triggers
inferred from cssom() only"]` and mark those rebuilds `authored-not-verified` rather than claiming them measured.

`scroll()` and `states(sel, ms)` are async and loop internally — one MCP call each, not one per step. Two
helpers ride along: `motion.fit(pairs)` re-fits a channel without re-sweeping, and `motion.scrollTargets()`
caches candidates at `window.__clone.state.scrollTargets`, overwritable before `scroll()`.

## M2 — keyframes and friends, verbatim

`cssom()` returns `keyframes`, `propertyRules`, `startingStyle`, `timelines`, `reducedMotionRules`, `usages`,
`transitionRules`, `viewTransition`, and a `supports` probe. Copy, do not paraphrase:

- `steps[].offset` is the browser-normalised `keyText` (`from` → `"0%"`; multi-selector steps stay
  `"0%, 100%"`). Never re-derive it.
- `steps[].decl` comes off the keyframe's own style block — one of the few places the CSSOM hands you
  **authored** values, so `translateY(2rem)` stays `2rem`. Emit as-is.
- `@property` is load-bearing, not decoration: an animated `<angle>` gradient silently does nothing without
  its registration. Emit every `propertyRules` block before the keyframes that depend on it.
- Reduced-motion keyframes are keyed `reduced:<name>` — ship both halves of the pair. A name defined twice
  collapses in this map, and `blocked[]` lists unreadable cross-origin sheets; for either, grep the recovered
  stylesheet text in `.clone/css/` (`references/assets.md`).

## M3 — transitions

`getComputedStyle` returns the resolved shorthand as comma-separated lists independently cycled to the length
of `transition-property`; index `i` of `duration` may come from a shorter list wrapping around.
`transitions()` zips them and groups by element role. Use `canonicalByRole`: the highest-count signature per
role becomes a foundation token (`--ease-*` plus a duration); only deviating elements get verbatim legs in
their section spec. Two things that are behaviour, not sloppiness, and must survive the copy:
`transition-property: all` transitions properties you later add — record it verbatim, do not "improve" it
into a property list; and `transition-behavior: allow-discrete` means the original animates
`display`/`overlay` — drop it and dialogs, popovers, and drawers pop instead of fade.

## M4 — `getAnimations()`, the call nobody makes

Highest-value read in motion extraction: it reports what the browser is actually running, already resolved,
regardless of which stylesheet or library produced it. The subclass alone identifies the mechanism.

| `constructor.name` | Produced by | Distinguishing member |
|---|---|---|
| `CSSAnimation` | a CSS `animation` property | `animationName` |
| `CSSTransition` | a CSS transition in flight | `transitionProperty` |
| `Animation` | `el.animate()` — Motion, framer-motion's WAAPI path, hand-rolled JS | `id` (often the library's label) |

`anims()` records per animation: `getTiming()` (authored `duration`/`delay`/`easing`/`iterations`/`fill`;
`duration` is the string `"auto"` when scroll-driven), `getComputedTiming().progress` (where in the curve
this sample sits), `getKeyframes()` (fully computed, camelCase, **used** values — the ground truth for
library animations that have no CSS anywhere), `pseudoElement`, `playState`, `playbackRate`, `replaceState`,
and `timeline.type` (`DocumentTimeline` | `ScrollTimeline` | `ViewTimeline`).

It only reports animations that exist **right now**, so call `anims('<label>')` at five moments and merge: at
load; at each scroll stop; mid-hover (§M7); right after the drawer toggle click
(`references/responsive.md`); and after `cdp:emulate({colorScheme:"dark"})` if there is a theme transition.
`paused` + `ScrollTimeline` is normal — driven by scroll, not time. `replaceState: "removed"` is evidence of
repeated re-triggering.

The payload collapses identical `(name, timing, keyframes)` sets into one entry with `groupCount` and
`staggerMs`: twelve cards with delays `0, 80, 160, 240…` are **one** spec plus `stagger: 80ms`, so the
section agent gets one line instead of twelve. `loops[]` (iterations `Infinity`) must be rebuilt as looping
CSS animations, never as one-shot reveals.

## M5 — library detection → rebuild implication

`libs()` probes globals *and* DOM signatures, because a bundled build leaves no `window.gsap`. Fill each
`motion.json.libraries[].rebuild` from this table; `evidence` comes from the payload.

| Library | Probe | Rebuild implication |
|---|---|---|
| GSAP | `window.gsap`, `.pin-spacer`, `[data-flip-id]` | rAF-driven, **invisible to `getAnimations()`**. The payload dumps `gsap.globalTimeline.getChildren()` with duration/ease/stagger. Port each tween to CSS/WAAPI, or keep GSAP with the same numbers. |
| ScrollTrigger | `window.ScrollTrigger`, `.pin-spacer` | `getAll()` gives `startPx`/`endPx`, the authored `start`/`end` strings, `scrub`, `pin`, `snap`, `toggleActions` — a complete spec, copy it. `scrub` ⇒ scroll-linked ⇒ `animation-timeline`. `toggleActions` ⇒ discrete ⇒ IntersectionObserver + class. `pin` ⇒ reproduce the `.pin-spacer` height or the layout jumps. |
| ScrollSmoother | `ScrollSmoother.get()`, `#smooth-wrapper` | scroll position is virtual and real `scrollY` stays ~0 — never derive progress from `window.scrollY`; `util.scrollTo` routes through it. Locomotive v3 does the same, and its `data-scroll-speed` attribute **is** the parallax spec: read it instead of fitting curves. |
| Lenis | `window.lenis`, `html.lenis` | keeps native scroll position, values just lag ⇒ `settle()` before sampling. Rebuild = install Lenis with the same `lerp`/`duration`/`easing` (`page()` dumps `lenisOptions`). |
| Motion / Motion One | `window.Motion`, `motion.animate` | WAAPI ⇒ fully visible in `anims()` and in the hook log with authored keyframes. Best case. |
| framer-motion | `[data-projection-id]`, `__FRAMER_MOTION__` | layout animations are FLIP measured at runtime; there is no static keyframe. Capture start/end rects at both states and rebuild with FLIP or `view-transition-name`. |
| Framer (builder) | `[class^="framer-"]` | class names are content-hashed and meaningless. Derive semantic names; do not mirror them. |
| AOS / ScrollReveal | `[data-aos]`, `[data-sr-id]` | the attributes are the whole spec. The pre-reveal state lives in the library's own CSS — take it from `cssom()`, because once you have scrolled past, the computed style shows only the revealed state. |
| Webflow IX2 | `[data-w-id]`, `Webflow.require('ix2')` | the payload dumps `store.getState().ixData` — interaction, trigger, easing, duration and target as JSON. Authoritative, skip empirical fitting. Capped at the first **60** `events` and **60** `actionLists`; `ix2.truncated:true` plus `ix2.warnings[]` says the store was bigger, and the remainder comes from `anims()`/`drain()`. |
| Swiper / Embla / Splide | `.swiper-initialized`, `.embla__viewport` | the payload reads Swiper's live `params`, including the `breakpoints` object, which is **responsive data that exists nowhere in the CSS**. Embla's options are JS-only: infer from geometry or accept a close equivalent and say so. |
| SplitText / Splitting | `.split-line`, `[data-splitting]` | text is split at runtime. The clone must split too — hard-coded spans break at other widths. |
| Lottie | `lottie-player`, `[data-animation-path]` | mirror the `.json`/`.lottie` byte-for-byte and re-mount the player. Never redraw it in CSS. |
| three.js / Vanta | `canvas[data-engine]`, `.vanta-canvas` | out of reach of CSS. Mirror the script if same-origin, else substitute and flag the gap. |
| Barba / Swup | `[data-barba]` | page transitions ⇒ `@view-transition { navigation: auto }` on the Next path; note the behavioural difference. |
| Alpine | `[x-transition]` | the attribute values encode enter/leave motion verbatim. |
| Tailwind `animate-*` | `[class*="animate-"]` | already CSS keyframes from §M2. Map to `--animate-*` theme vars. |
| anime.js · Rellax · KeenSlider (probed) · Rive · scrollama · VanillaTilt (**not** probed by `libs()`) | `window.anime`, `[data-rellax-speed]`, `.keen-slider` / for the last three check by hand: `@rive-app` + a `.riv` request, `.step` elements driven by many IO thresholds, `[data-tilt]` or `el.vanillaTilt` | anime v3 is rAF ⇒ fit it (§M6); Rive is a runtime canvas ⇒ mirror the `.riv` and re-mount, never reconstruct; scrollama is IO + steps ⇒ rebuild from `drain().observers`; VanillaTilt is a pointer-driven 3D transform ⇒ 20 lines of your own JS, no dependency |

If `libs().smoothScrollHijack` is true, **every** later measurement pass must go through
`window.__clone.util.scrollTo` + `settle` (the payload does), and the responsive sweep must re-check that
scroll resets to 0 after each viewport change — these libraries frequently do not.

## M5b — builder-native manifests: read the spec instead of inferring it

Builders ship their motion spec into the page as data, and when one is present it **outranks everything else
in this file** — authored source, not a reconstruction. Framer carries the appear-animation spec and the
breakpoint list as inline script tags; pass this with `filePath: .clone/raw/motion-builder.json`:

```js
() => {
  const out = { manifests: {}, found: [] };
  for (const s of document.querySelectorAll('script[id^="__framer__"], script[type^="framer/"]')) {
    out.found.push({ id: s.id || null, type: s.type || null, bytes: s.textContent.length });
    try { out.manifests[s.id || s.type] = JSON.parse(s.textContent); }
    catch (e) { out.manifests[s.id || s.type] = { unparsed: s.textContent.slice(0, 4000) }; }
  }
  out.appearTargets = [...document.querySelectorAll('[data-framer-appear-id]')].slice(0, 400).map((el) =>
    ({ appearId: el.getAttribute('data-framer-appear-id'), path: window.__clone.util.cssPath(el) }));
  return out;
}
```

`__framer__appearAnimationsContent` holds the per-`appearId` entrance spec (`transform`/`opacity` from-values,
`duration`, `delay`, an `ease` array or spring `stiffness`/`damping`); `__framer__breakpoints` holds the list
`references/responsive.md` §R1 wants. Join `appearTargets[]` on `data-framer-appear-id` and entrance motion is
per-element with zero fitting. **Enumerate, never hard-code the ids** — builders rename them between versions.

| Builder | Manifest | Read it with |
|---|---|---|
| Framer | `script#__framer__appearAnimationsContent`, `script#__framer__breakpoints`, `[data-framer-appear-id]` | the probe above |
| Webflow | `Webflow.require('ix2').store.getState().ixData` | `libs()` already dumps it (§M5) |
| Elementor | per-widget `[data-settings]` JSON: `animation`, `animation_delay`; breakpoints in `elementorFrontendConfig.responsive.breakpoints` | parse the attribute / read the global |
| Swiper | live `swiper.params`, including `breakpoints` | `libs()` |
| AOS · ScrollReveal · Alpine · Locomotive | `data-aos*`, `data-sr-id`, `x-transition`, `data-scroll-speed` | the attributes *are* the spec (§M5) |

Set `evidence` to the manifest path (`motion-builder.json#manifests.__framer__appearAnimationsContent`) and
skip §M6 fitting for whatever it covers. Builder class hashes stay off-limits — never mirror them.

## M5c — hand-rolled motion: read their bundle

M5b covers **builders**. When `libs()` names GSAP/ScrollTrigger/Lenis/SplitType/Barba/Locomotive
and no builder manifest exists, the site is bespoke and its motion configuration lives in its own
JS. **Read `references/motion-source.md` now.** It is a full pass: mirror the first-party bundle,
recover the author's module boundaries, and extract the eleven things that are otherwise guessed —
smooth-scroll options, scroll thresholds, the per-reveal duration/stagger/ease map, trigger
offsets, the parallax rule, split selectors, the resize behaviour, slider configs, page
transitions, the loader.

Runtime introspection (M4, M5) is necessary but **not sufficient** here, because it only sees what
exists at the moment you look:

| invisible at runtime | but plainly in the source |
|---|---|
| a `once: true` trigger that already fired — it kills itself | the timeline definition |
| a timeline for a section still below the fold | the same |
| the authored from/to of a `scrub` tween (you see current progress) | `fromTo(...)` |
| a threshold inside `if (scroll > N)` that sets a class or data attribute | the literal `N` |
| a resize handler's behaviour | the handler |

Two traps that produce confidently wrong rebuilds, both expanded in that file:

- **`IntersectionObserver` may be entirely absent.** On a GSAP site the reveals are ScrollTrigger
  timelines. One real 3 MB bundle contained exactly **two** `IntersectionObserver` references,
  both incidental, against 159 for `ScrollTrigger`. Inferring IO from behaviour and choosing your
  own thresholds makes every reveal fire at visibly the wrong moment. Check the ratio before
  assuming.
- **Copy the behaviour, not the intent.** When their handler looks wrong, reproduce it anyway. A
  real resize handler reverted every text split and did not rebuild — so after any viewport change
  that site has no line masks until the next navigation. The "obviously correct" rebuild gave the
  clone masks where the origin had none, worth **+14 to +63 px** of section height at 1024/390 and
  failing three previously-passing sections.

## M6 — scroll-linked effects

**Native first.** If `cssom().timelines[]` is non-empty, the rebuild is a copy: `scroll-timeline-name` /
`view-timeline-name` on the source, `animation-timeline` + `animation-range` on the animated element,
`timeline-scope` when they are not ancestor-related. `anims()` confirms it at runtime — `timeline.type` is
`ScrollTimeline`/`ViewTimeline`, `playState` is `paused`, `timing.duration` is `"auto"`.

**Otherwise characterise empirically.** The orchestrator call sequence, in full:

```
cdp:evaluate_script  function: () => { window.__clone.state.scrollTargets = ["<sel>","<sel>"]; return 1; }   // optional; else scrollTargets() derives them
cdp:evaluate_script  function: () => window.__clone.motion.scroll(21)      filePath: .clone/raw/motion-scroll.json
```

One call, 21 internal stops. At each stop the payload scrolls with library-aware `scrollTo`, waits for
`settle()` (lerp/inertia libraries keep easing for many frames after the position is set — sampling
immediately gives a point that is *not on the curve*), then reads `transform` decomposed into `ty/tx/sx/rot`,
`opacity`, blur, the raw strings, and any `--progress`-style custom property.

Two x-axes are recorded and both fitted: `p` = page progress (0–1 of max scroll), for header and global
effects; `cover` = the element's own progress through the viewport, defined exactly as the CSS `view()`
timeline `cover` range, so it is viewport-height independent and portable. The better `r2` wins and is
reported as `driver`. `fit` trims the flat head and tail — those bounds **are** the `animation-range`
offsets — regresses the active region, and emits `linearEasing`, which is what makes this exact rather than
approximate: CSS `linear()` takes an arbitrary piecewise-linear easing, so a 21-point sample of any monotone
curve becomes a one-line easing. Check `cssom().supports.linearEasing` first.

| Measured shape | Rebuild |
|---|---|
| `kind: linear`, `driver: cover`, range ≈ `{0,1}` | `animation-timeline: view(); animation-range: cover 0% cover 100%;` + two keyframes |
| `kind: linear`, range `{0.1, 0.45}` | same, with `animation-range: entry 10% cover 45%` |
| `kind: eased` | same timeline + `animation-timing-function: <linearEasing>` |
| `driver: page`, target is the header | `animation-timeline: scroll(root block)` |
| `kind: nonMonotone` (pin + multi-phase scrub timeline) | multi-keyframe `@keyframes` at the measured `keyframeOffsets`; reproduce the pin (sticky wrapper of measured height) before animating |
| toggles once and stays (AOS-style reveal) | **not** scroll-linked. IntersectionObserver with the `rootMargin`/`threshold` from `drain().observers`, toggling a class; base state from `cssom()` |
| `docHeightDrift: true` in the sweep output | virtualised or infinite-scroll content. Cap the sweep, mark the fits approximate, list it in `motion.json.gaps` |

## M7 — hover, focus, active

`el.dispatchEvent(new MouseEvent('mouseover'))` does **not** make `:hover` match — CSS `:hover` is driven by
the browser's hit test against real pointer input. `cdp:hover` issues a real CDP mouse event, so it captures
both the CSS state and any JS-driven one. Conversely the pointer route cannot reach `:focus-visible`,
`:active`, `:disabled`, `:checked`, `:popover-open`, or `::selection`. Run both routes.

Pointer route, per element — budget 8–15 (primary button, nav link, card, input, menu toggle, footer link):

```
cdp:take_snapshot    filePath: .clone/raw/snapshot-hover.txt        → harvest the uid
cdp:evaluate_script  function: () => window.__clone.motion.states('<sel>')       // rest, cached
cdp:hover            uid: "<uid>"
cdp:evaluate_script  function: () => window.__clone.motion.states('<sel>', 90)   // mid-flight
cdp:evaluate_script  function: () => window.__clone.motion.states('<sel>', 400)  // settled
cdp:hover            uid: "<uid of body or a neutral element>"                   // reset, or hover leaks
```

The rest read caches its snapshot on `window.__clone.state.rest`, so the hover reads come back as a `delta`
map of `{from, to}` per changed property — base plus `::before` plus `::after`, a state no screenshot shows.
Take the mid-flight read too: its `anims[]` holds live `CSSTransition` objects whose `keyframes` state the
exact from/to pair the browser computed, which resolves ambiguous cases like `box-shadow` interpolation. uids
go stale on every DOM mutation — re-snapshot before each uid-bearing call.

`:focus-visible` needs `cdp:press_key({key:"Tab"})` to walk focus (a click does not produce it), then the same
`states('<sel>')` read. `:active` needs a held mouse button, which the MCP surface does not offer: take it
from the CSSOM route and mark it `authored-not-verified`.

Everything the pointer cannot reach is why `interactive[].state` is a **wider** vocabulary than the four pointer
states: `hover | focus | focus-visible | focus-within | active | target | checked | indeterminate | disabled |
placeholder-shown | invalid | user-invalid | valid | autofill | open | popover-open | modal | fullscreen`. The CSSOM
route names all of them (`norm()` in `scripts/extract-motion.js`), and they arrive `authored-not-verified` because
nothing in the MCP surface can put a real `<input>` into an invalid or checked state. One caveat to state in
`report.md`: a state rule whose base selector currently matches **zero** elements is dropped, so a `:checked` style
on a page whose radios are all unchecked is genuinely unmeasured — say so rather than inventing it. Forms carry more
than states; the field-level contract is `references/agent-brief.md` §5a.

That route (`states()`, no argument) is exhaustive
and returns `common[]` — one hover treatment shared by 40 selectors is a **token**, not 40 specs. Individual
`rules[]` go to the owning section agent verbatim, including `media`: a rule gated behind
`@media (hover: hover)` must stay gated, or the clone gets stuck hover states on touch.

## M8 — `prefers-reduced-motion`

No MCP tool forces it. `cdp:emulate` covers `viewport`, `colorScheme`, `userAgent`, `networkConditions`,
`cpuThrottlingRate`, `geolocation`, `extraHttpHeaders` — nothing else. The Playwright fallback has no
`emulateMedia` tool either; do not claim it does. Two real routes: `cssom().reducedMotionRules` plus the
`reduced:` keyframe pairs are the authored variant verbatim (fill `reducedMotion.authored` and
`.conditions` from them), and `--force-prefers-reduced-motion` is a real Chrome launch switch worth
mentioning once if the user wants a runtime diff. `page().media.reducedMotion` reports which variant the
current session is seeing.

Non-negotiable: if the original has zero reduced-motion rules (common), the clone still ships one — fidelity
does not extend to reproducing a missing accessibility affordance. Every non-decorative animation gets a
`@media (prefers-reduced-motion: reduce)` branch collapsing transforms to a crossfade or an instant state
change, loops get `animation-play-state: paused`, `reducedMotion.clonePlan` records it, and `CLONE-REPORT.md`
gets one line calling it a deliberate deviation.

## M9 — merging into `motion.json`

`page()` fills the scroll chrome — `scrollBehavior`, `overscroll`, `snapType`, `snapChildren`,
`scrollPaddingTop`, `viewTransitions`, `smoothScrollLib` — plus `pinned[]` (sticky/fixed elements, the #1
clone miss after motion itself) and the `units` block whose `scrollbar` value the responsive sweep needs.
SPEC §D3 declares `"page"` twice in motion.json (as {id,url} and again as the scroll-chrome object);
duplicate JSON keys are lossy, so `page:{id,url}` stays (matching D4) and the scroll-chrome block is named
`pageScroll`, which is what `motion.page()` returns.
Then split the merged truth per section, keyed by `sectionId`, using each section's `box` from
`.clone/sections.json` to decide ownership. `bySection.<id>` carries `entrance`, `scrollLinked`, `hover`,
`loops`, `reducedMotion`, `gaps` — numbers, never "make it feel smooth". **`evidence` is required on every
`bySection` entry**, as `"<raw file>#<json path>"` (the scroll payload emits its own, e.g.
`motion-scroll.json#samples[].els[3].ch.ty`); an entry without evidence is deleted, not shipped. That is how
the review pass checks a section agent's work without re-measuring.

## M10 — rebuild recipes: when plain CSS is enough, and when it is not

Default to CSS. Reach for a dependency only when the measurement says you must, and say so in the report.

| Measured original | Ship |
|---|---|
| entrance reveals with a uniform `staggerMs`, one-shot | CSS `@keyframes` + `animation-delay: calc(var(--i) * <stagger>ms)`, triggered by an IntersectionObserver class toggle with the captured `rootMargin`/`threshold` |
| monotone scroll-linked transform/opacity | `animation-timeline: view()` / `scroll()` + `animation-range` + the fitted `linearEasing`. Zero JS. |
| hover/focus deltas | plain CSS state rules with the measured transition legs, gated by the captured `@media (hover: hover)` |
| infinite marquee / pulse / gradient drift | CSS keyframes with `iterations: infinite`, registered via `--animate-*`; duplicate the track for a seamless loop |
| `nonMonotone` pinned scrub timeline, multi-phase | keep **GSAP + ScrollTrigger** with the captured `startPx`/`endPx`/`scrub`/`pin`. A CSS reconstruction of a pinned multi-phase timeline is a rewrite, not a clone. |
| physics: spring, `lerp`, velocity-coupled skew, drag inertia | keep **Motion** (`spring` options) or GSAP. Fitting a curve to a velocity-coupled effect produces a plausible lie. |
| smooth-scroll hijack that the design depends on (parallax phasing, marquee coupling) | keep **Lenis** with the measured `lerp`/`duration`/`easing` from `page().lenisOptions` |
| canvas/WebGL/Lottie | mirror the asset, re-mount the player, and list it as not-reconstructed |

Every dependency added this way goes into `requests.json` as `{"kind":"dependency"}`
(`references/agent-brief.md`) and gets one line in `CLONE-REPORT.md` stating what it reproduces and what a
CSS-only version would have lost. Never add an animation library the original did not need.

### The reveal-on-scroll pattern, in full

The most-copied effect on the web and the easiest to ship broken. **Hard rule: the default state is the final,
visible state, and the hidden state exists only once JS is alive** — so the page never ships blank to a
crawler, a link-preview fetcher, a headless renderer, or a JS-disabled user, which is exactly what a naive
`opacity:0` + IntersectionObserver clone does. Gate the hidden state behind a class only JS can add:

```css
[data-reveal] { transition: opacity var(--reveal-dur, 600ms) var(--ease-out),
                            transform var(--reveal-dur, 600ms) var(--ease-out);
                transition-delay: calc(var(--i, 0) * var(--reveal-stagger, 80ms)); }
.motion-armed [data-reveal]:not(.is-revealed) { opacity: 0; transform: translateY(24px); }
@media (prefers-reduced-motion: reduce) {          /* crossfade, no travel, no stagger */
  .motion-armed [data-reveal]:not(.is-revealed) { opacity: 0; transform: none; }
  [data-reveal] { transition: opacity 200ms linear; transition-delay: 0ms; }
}
```

```js
// Arm and sweep in ONE synchronous block: the browser never paints the hidden state
// for content already on screen. rootMargin/threshold come from drain().observers.
document.documentElement.classList.add('motion-armed');
const io = new IntersectionObserver((entries, obs) => {
  for (const e of entries) if (e.isIntersecting) { e.target.classList.add('is-revealed'); obs.unobserve(e.target); }
}, { rootMargin: '0px 0px -10% 0px', threshold: 0.15 });
for (const el of document.querySelectorAll('[data-reveal]')) {
  const r = el.getBoundingClientRect();
  if (r.top < innerHeight && r.bottom > 0) el.classList.add('is-revealed');  // in view at mount: show, never animate
  else io.observe(el);
}
```

`--i` per child comes from the measured `staggerMs`, `--reveal-dur` and the easing from `anims()`/`cssom()`.
Run it on `DOMContentLoaded` or from a client-component effect, but keep arming and the in-view sweep in the
same synchronous pass or above-the-fold elements flash; below-fold elements are hidden before they are ever
painted, which is why the ordering works. The CSS branch already covers reduced motion, so skipping the arm
when `matchMedia('(prefers-reduced-motion: reduce)').matches` is belt-and-braces, not a second path.

## Motion fidelity gate

Structural checks, run before you claim motion parity. Numeric tolerances for every gate live in
`references/assembly.md`; do not restate or invent them here.

1. Every `bySection` entry has non-empty `evidence`; entries without it are deleted, not shipped.
2. Every keyframe name the clone uses exists with identical step offsets, and every `@property` block a
   keyframe depends on was emitted.
3. `transitions().canonicalByRole` re-measured on the clone matches the original's map, role for role.
4. `anims()` on the clone yields the same group count, the same `iterations: Infinity` set, and matching
   `staggerMs` per group.
5. Every `scrollLinked` entry with `fit.kind` `linear` or `eased` is reproduced by a timeline or a fitted
   easing — not omitted, not downgraded to a fade.
6. Hover: the *set of changed properties* matches per probed element, and gated rules stayed gated.
7. `reducedMotion.clonePlan` is implemented and the clone's reduced-motion branch parses.
8. Everything unverifiable is labelled rather than silently upgraded — `:active` `authored-not-verified`,
   reduced-motion runtime unmeasured, rAF physics fitted, canvas/WebGL/Lottie mirrored or substituted — and
   listed in `motion.json.gaps` and `CLONE-REPORT.md`.
