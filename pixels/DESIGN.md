# Pixel/Flow design system

Pixel/Flow is a dark, monospaced, grid-locked interface language where **surfaces don't appear, they assemble**: every state change is a field of square cells switching on with a sweep, a little jitter, and a faster, reversed exit.

The whole system lives in one file, [`pixel-flow-design-system.html`](pixel-flow-design-system.html): tokens and component CSS in the `<style>` block, markup demos in `<main>`, and every behaviour in a single script wrapped in one IIFE. This document explains how to build new interfaces with it, for people and for LLM agents. When this guide and the HTML disagree, the HTML wins; fix this file.

> **To view it:** run `python -m http.server 8000` inside `pixels/` and open `http://localhost:8000/pixel-flow-design-system.html`. Some parts (fonts, the Openverse image) need the network.

---

## 1. Rules first (read this if you read nothing else)

These are the non-negotiables. An interface that breaks one of them is not Pixel/Flow.

1. **Everything sits on the 12px unit (`--u`).** Sizes, padding, gaps, line heights and component heights are whole units (or 6px half-units for fine detail). Never write `padding: 10px` or `height: 40px`.
2. **Four colours, never a new hue.** Void (black), ash (grey), bone (warm off-white) and signal (orange), plus their listed tints. No greens for success, no blues for links, no reds for errors.
3. **Signal is reserved** for action, focus and errors (and the "leading edge" of a motion). If something orange isn't one of those, make it bone or ash.
4. **State changes assemble.** Fills, highlights and surfaces come in through a **hatch** (a canvas of cells), not a CSS fade or slide. Reuse the hatch engine rather than inventing a transition.
5. **Off is a resting cell, not nothing.** The background field, muted cells and dim frames keep the grid legible even when nothing is active.
6. **Type is mono only**, set in DM Mono (with PP Supply Mono preferred if installed). No second family.
7. **Square corners, 1px rules, no shadows for depth.** Borders are `inset` box-shadows of 1px; there is no `border-radius` anywhere.
8. **Motion respects `prefers-reduced-motion`.** CSS transitions are zeroed globally and the hatch engine jumps to its end state. Don't add motion that bypasses either.
9. **Accessible by default.** Real roles and ARIA states drive the visuals (`aria-checked`, `aria-selected`, `aria-expanded`), focus is always visible as a 1px signal outline, and every overlay closes with Esc.

### For LLM agents

- Copy markup patterns from the existing demos in the HTML. Every component in section 7 has a live example; search for its class name.
- The script is a single closure. `toast`, `Dialog`, `Field`, `initSeg`, `recolorHatches` and friends are **not globals**. Add new behaviour **inside** that IIFE (before its closing `})();`), or deliberately expose an API on `window`.
- Elements are wired once at load (`$$('[data-hatch]').forEach(initHatch)`, `$$('[data-roll]')`, `$$('[data-arrow]')`, …). Markup you inject later must be initialised by calling the same functions, as `toast()` does for its own elements.
- Test in a browser over `http://` (not `file://`), and verify geometry with DOM and canvas measurements, not only screenshots.

---

## 2. Foundations

### 2.1 Colour

All values are CSS custom properties on `:root` (see the `Tokens` banner).

| Token | Hex | Role |
|---|---|---|
| `--void` | `#000000` | Ground. Page background, text on light fills |
| `--void-900` … `--void-500` | `#080808` → `#30302f` | Depth: surfaces, rules, idle cells. `--void-500` is the standard 1px rule |
| `--ash-700` | `#5c5c5d` | Placeholders, meta, disabled text (≈3:1 on black, so never body copy) |
| `--ash` | `#888889` | Secondary text, idle glyphs, resting cells (≈5.9:1) |
| `--ash-300` | `#b0b0b1` | Field labels, prose, quieter primary text |
| `--bone-700` | `#a8a296` | Focus borders, the input focus bar, selected outlines |
| `--bone-500` | `#cdc6b9` | Fills inside tracks and meters |
| `--bone` | `#ece5d8` | Primary ink: text, fills, glyphs, the default hatch colour |
| `--bone-100` | `#f6f2ea` | "Done" or success fill (brighter than bone, still not a new hue) |
| `--signal-900` … `--signal-300` | `#3b1609` → `#ff8757` | Error surfaces, live tags, accents on signal fills |
| `--signal` | `#e85622` | Action, focus outline, error, the leading edge |

Usage patterns:

- **Status without new hues:** idle is bone, busy is ash, done is bright bone (`--bone-100`), and fail is signal. Success is told by its glyph, label and the wipe it lands with, not by colour (see the stateful submit button).
- **Text on light fills** (bone or signal hatches) is `--void` at **weight 500**, because dark-on-light reads thinner. Mono advance widths don't change with weight, so nothing reflows.
- **Translucent surfaces** use black at 62% (`rgba(0, 0, 0, .62)`) for panels and cards, and 90% for the header, so the field shows through, muted.

### 2.2 Type

One family: `--font-mono` (`"PP Supply Mono", "DM Mono", ui-monospace, monospace`), weights 300, 400 and 500.

| Use | Size / line height | Notes |
|---|---|---|
| Body | 13 / 24px | Default on `body` |
| `.t-label` | 11 / 12px, `.06em`, uppercase | Buttons, nav, brand |
| `.t-micro` | 10 / 12px, `.08em`, uppercase | Kickers, field labels, help text, panel heads |
| Section `h2` | 32 / 36px, `-.02em` | Inside `.sec-head` |
| Panel or card `h3` | 20 / 24px, `-.01em` | |
| Lede | 16 / 24px, `-.01em` | `.hero__lede` |
| Stat | 32 / 36px | `.stat__value`, with a `<small>` unit in ash |
| Big counter | 48 / 48px, signal | `.loader__count` |

Helpers: `.muted` (ash), `.signal`, `.num` (tabular figures), `.prose` (ash-300 paragraphs), `.sr-only`.

Line heights are always 12 or 24px multiples, so text blocks stay on the field's rows.

### 2.3 Spacing and size

- `--u: 12px` is the base unit; `--cell: 6px` is the half unit used for hatch cells and fine insets.
- Heights: inputs, normal buttons and list rows are 48px (4u); small buttons, segments and menu options are 36px (3u); large buttons are 60px (5u); switches, checks, radios, chips, tags and tracks are 24px (2u).
- Gaps: `.stack` and `.row` use 2u; `.row--tight` uses 1u; label-to-control is 6px (`.px-field`, `.px-input`).
- Panels are padded 2u. Their content uses negative margins (`margin: 0 calc(var(--u) * -2)`) when it should bleed to the panel edge (lists, media).

### 2.4 Motion tokens

| Token | Value | Use |
|---|---|---|
| `--dur-cell` | 220ms | One cell's fade in a hatch (override per host with `--px-cell-dur`) |
| `--dur-exit` | 320ms | Exits and collapses |
| `--dur-base` | 420ms | Standard transitions |
| `--dur-roll` | 525ms | Label roll and glyph travel |
| `--ease-out` | `cubic-bezier(.22, 1, .36, 1)` | Arrivals |
| `--ease-snap` | `cubic-bezier(.625, .05, 0, 1)` | Rolls, knobs |
| `--ease-cell` | `cubic-bezier(.66, 0, .34, 1)` | Per-cell fades (mirrored in JS as `easeCell`) |

Non-hatch motion prefers **stepped** timing (`steps(4, end)` for menus, toast slots and chips; `steps(8)` for link underlines; `steps(24)` for the skeleton shimmer), so things move in pixels rather than gliding.

---

## 3. Layout

### 3.1 Page skeleton

```html
<canvas id="field" aria-hidden="true"></canvas>   <!-- the background grid, fixed, z 0 -->
<header class="header">…</header>                  <!-- fixed, 4u tall, z 10 -->
<main class="page" id="top">                       <!-- z 1 -->
  <section id="components">
    <div class="wrap">
      <header class="sec-head" data-quiet>
        <span class="sec-head__idx">02 /</span>
        <h2>Components</h2>
        <p>One-paragraph intro in ash.</p>
      </header>
      <div class="grid">
        <div class="panel s-6" data-panel data-snap>…</div>
      </div>
    </div>
  </section>
</main>
<div class="px-toasts" id="toasts" role="region" aria-label="Notifications" aria-live="polite"></div>
<div class="px-tip" id="px-tip" role="tooltip" …></div>
<!-- <dialog class="px-dialog">s go here -->
```

Sections are spaced 12u apart (`section { padding-top }`). The z-index scale is: page 1, header 10, toasts 20, menus 30, tooltip 40, and dialogs in the browser's top layer.

### 3.2 Grid and breakpoints

`.wrap` is the content column; `.grid` lays out `--cols` columns with a 2u gap. Spans: `.s-12`, `.s-9`, `.s-8`, `.s-6`, `.s-4`, `.s-3`.

| Viewport | Columns (`--cols`) | `.wrap` width | Span remapping |
|---|---|---|---|
| > 1175px | 12 | 1128px | as written |
| ≤ 1175px | 8 | 744px | 12, 9 and 8 become 8; 6, 4 and 3 become 4 |
| ≤ 791px | 6 | 552px | 12 to 6 become 6; 4 and 3 become 3 |
| ≤ 599px | 4 | 100% − 2u | everything becomes 4 |

`--cols` is the **layout** column count and is inherited everywhere. Don't reuse the name for a component's own column count (the 2D segment uses `--seg-cols` for exactly this reason).

### 3.3 Surfaces

- **`.panel`**: a translucent black surface with a 1px `--void-500` frame, ash corner ticks (top left and bottom right), and 2u padding. Start it with a `.panel__head.t-micro` row (`<b>` title in bone on the left, a meta `<span>` in ash on the right, 2u tall, bleeding to the edges).
- **`.stack`**: a vertical grid with a 2u gap. **`.row`**: a wrapping flex row with a 2u gap (`.row--tight` for 1u). **`.sub.t-micro`**: a subheading with a rule that runs to the right edge.
- **Surface attributes** (read by the field and the snapper):

| Attribute | Effect |
|---|---|
| `data-panel` | The field's cells under this element drop to resting size, with a two-cell falloff around the edge |
| `data-quiet` / `data-quiet=".6"` | A lighter mute for text blocks (default .85 of a panel's) so headings stay legible over the field |
| `data-snap` | Height is rounded up to whole units by a ResizeObserver (`snapRO`), so the field runs straight through the layout |

Buttons (`.px-btn`) are width-snapped to whole units by `snapButtons(root)`. It skips hidden buttons (width 0); call `snapButtons(el)` again after showing hidden content.

---

## 4. The field (background grid)

`#field` is a full-viewport canvas drawing one square per 12px cell across the whole document, aligned to `.wrap`'s left edge. It's implemented as `const Field = (() => { … })()` under the `Field: page-wide pixel grid` banner.

- Cell sizes are driven by drifting, domain-warped value noise (`flow()`). Cells rest at 2px and swell up to 9px.
- The pointer leaves a decaying heat trail that turns cells signal; clicking empty space sends a ripple. Clicks on controls, panels, the header, toasts and dialogs are excluded.
- Under `[data-panel]` and `[data-quiet]` surfaces, cells are muted (see 3.3).
- **`data-glyph`** renders text *as field cells* (the hero wordmark). Attributes: `data-glyph="PIXEL/FLOW"`, `data-glyph-alt="PIXEL|FLOW"` (`|` breaks lines; used when narrower than 90 cells), `data-glyph-mask` (how strongly the area behind is muted), `data-glyph-align`, and `data-glyph-stroke`. Put an `.sr-only` heading inside for screen readers.
- API (inside the script): `Field.relayout()` after layout changes that move surfaces (it's debounced, and called by `snapRO` automatically); `Field.setRunning(bool)` to pause the animation. The header switch `#field-toggle` uses it, and it starts off under reduced motion.

Colours are drawn from a fixed list of palette tints (`COLORS` inside `Field`). If you add field effects, pick from that list.

---

## 5. The hatch engine

A **hatch** is a `<canvas>` with **one canvas pixel per cell**, scaled up with `image-rendering: pixelated`. Each cell's alpha animates from its current value to the target with a per-cell delay:

> **delay = position × sweep + random × jitter.** The exit mirrors the order and runs about 25% faster (delays × .78, fade × .8).

Code lives under the `Hatch: per-cell delays, one canvas pixel per cell` banner.

### 5.1 Making something a hatch host

Add `data-hatch` to an element. At load, `initHatch(host)` prepends `<span class="px-hatch"><canvas></canvas></span>`, sizes it to the host, and starts observing the host's size.

The host must be a positioning and clipping context, and its real content must sit above the hatch:

```css
.my-thing { position: relative; overflow: hidden; --hatch: var(--bone); }
.my-thing > :not(.px-hatch) { position: relative; z-index: 1; }
```

### 5.2 Attributes

| Attribute | Default | Meaning |
|---|---|---|
| `data-hatch` | `x` | Sweep order: `x` (left to right), `y` (top to bottom), `diag`, `center` (outward), `scatter` (random), `entry` (outward from an entry point, retracting toward an exit point) |
| `data-hatch-cell` | `6` | Cell size in px. Use 4 for tiny controls, 6 for controls, 12 for surfaces (12 matches the field's cell size) |
| `data-hatch-sweep` | auto | Total sweep time in ms. Auto is `clamp(steps × 7, 120, 520)` |
| `data-hatch-jitter` | `55` | Random extra delay per cell, in ms |
| `data-hatch-tone` | `0` | Fraction of cells painted with `--hatch-b` instead of `--hatch` (texture) |
| `data-hatch-breathe` | off | Lit cells dim by up to this amount on their own slow cycles. Present (even `"0"`) means breathing can be switched on later |
| `data-hatch-entry` / `data-hatch-exit` | `0 .5` / `1 .5` | Default entry and exit points (`"x y"`, 0–1) for `entry` mode |
| `data-hatch-group` (on an ancestor) | | Neighbouring hovers hand off quickly instead of replaying the full sweep (menus, lists) |
| `data-hover` | | Hover lights the hatch (fine pointers only), rolls `data-roll` labels, moves `data-arrow` glyphs, and makes it a press host |

CSS custom properties on the host (re-read on every state change):

| Property | Meaning |
|---|---|
| `--hatch` | Cell colour (any palette token) |
| `--hatch-b` | Second colour, for toned cells |
| `--hatch-rest` | Alpha when "off" (default 0; `.px-hatch--rest` sets .14) |
| `--px-cell-dur` | Per-cell fade duration (defaults to `--dur-cell`) |

### 5.3 When is a hatch "on"?

A hatch is on when its `.px-hatch` layer matches **`HATCH_ON`**, a selector list near the top of the hatch section. It includes:

- `.is-on > .px-hatch`
- `[aria-checked="true"] > .px-hatch`
- `[aria-selected="true"]:not([role="option"]) > .px-hatch`
- `:focus-visible > .px-hatch`
- `[data-hover]:hover > .px-hatch` (fine pointers only)
- Component-specific entries for inputs, the nav, dropdowns, the expand button and dialogs.

On-states are re-checked on the next frame (`queueHatchSync()`) after `pointerover`, `pointerout`, `focusin` and `focusout`, and after any change to `class`, `aria-checked` or `aria-selected` anywhere in the body (a MutationObserver).

So, to drive a hatch:

- **Use what's already there:** toggle `.is-on` on the host, or rely on ARIA state or focus.
- **New kinds of state:** add a selector to `HATCH_ON`, ideally with a child combinator (`.my-thing.is-open > .my-thing__fill > .px-hatch`). If your state lives in another attribute (such as `data-state`), call `queueHatchSync()` yourself after changing it.

### 5.4 Behaviour you get for free

- **Interruptions reverse at once.** A cell caught mid-fade turns around immediately (with a shortened ease-out) rather than finishing first; settled cells follow the normal sweep, starting from the first one that has to move.
- **Group handoffs.** Inside `[data-hatch-group]`, moving from one lit member to the next uses a compressed sweep (`HANDOFF`), so menus feel instant after the first hover.
- **Press feedback.** Pressing (`pointerdown`, or Enter, or Space on buttons) any `a` or `button` snaps its press hosts (`PRESS_HOSTS = '[data-hover], .px-xbtn__panel, .px-xbtn__icon'`) to full in a darker tone (`pressTone`), and adds `.is-pressed` to the control. On release, a square ripple (`sweepFront`) runs out from the press point back to the normal colour. Glyphs nudge 2px along their direction, or lean, while `.is-pressed`.
- **Reduced motion.** With `REDUCE` set, hatches jump straight to their end state.

### 5.5 Functions you'll call

| Function | Use |
|---|---|
| `initHatch(host)` | Wire a host added after load (build, observe resize, track for sync) |
| `buildHatch(host, force)` | (Re)build the canvas for the host's current size. It skips zero-size (hidden) hosts; the ResizeObserver rebuilds them when they appear |
| `setHatchState(st, on, now, handoff)` | Low-level: animate one hatch state (`host._hatch`) on or off. Prefer toggling classes |
| `setEntry(host, [x, y])` / `setExit(host, [x, y])` | Aim an `entry`-mode hatch, for example at the pointer or at a trigger |
| `localPoint(host, event)` | A pointer position as 0–1 host coordinates |
| `recolorHatches(el)` | After changing `--hatch` (error state, stateful button), re-read colours for every hatch inside `el` and wipe the new ones in from the left |
| `sweepFront(states, x, y, ahead)` | One square colour front across several hatches from a viewport point (used by press release and recolouring) |
| `queueHatchSync()` | Re-evaluate `HATCH_ON` on the next frame |

---

## 6. Label roll and glyph motion

- **`data-roll`** on a label turns its text into two stacked copies. In an active state (hover on `[data-hover]`, `:focus-visible`, `.is-on`, `.is-pressed`), the second copy (`--on` colour, weight 500) rolls up over `--dur-roll`.
- **`data-arrow="→"`** builds a two-copy glyph track. In an active state the glyph **moves the way it points**: `→ ← ↑ ↓ ↗ ↘ ↙ ↖ ▸` travel along `--dx` and `--dy`, while `↺ ↻ × + −` stay put, rotate by `--spin` and cross-fade (`--fade`). While pressed, glyphs nudge 2px, or lean by `--lean`. Add a new glyph by giving it `--dx`/`--dy` (or `--spin`/`--fade`/`--lean`) next to the existing `[data-arrow="…"]` rules.
- To change a glyph at runtime, set `el.dataset.arrow` **and** the text of both `.arrow > span` children (see the toast position control).

The roll and arrow rules use **descendant** selectors (`.is-on .roll__track`, `.is-on .arrow`). Never put `.is-on` on a container that holds buttons, or every button inside will flip to its active label. Use a dedicated class plus a child-combinator `HATCH_ON` entry instead, as dialogs do with `is-dimmed` and `is-raised`.

---

## 7. Components

Each entry gives the minimal markup; the HTML demo shows every variant. Search the file for the class name.

### 7.1 Buttons

**`.px-btn`**: a hatch-filled button with a label and an optional square glyph cell.

```html
<button type="button" class="px-btn" data-hatch="x" data-hover>
  <span class="px-btn__label" data-roll>Enter website</span>
  <span class="px-btn__icon" data-arrow="↗"></span>
</button>
```

- **Variants:** default (bone line and fill); `--signal` (the primary action); `--solid` (a bone surface with a signal hatch, for "continue" or confirm); `--quiet` (dark rule, ash text, the cancel or dismiss style); `[disabled]` (diagonal hatching, no hatch).
- **Sizes:** `--s` (36px), default (48px), and `--l` (60px).
- **Label only:** omit `.px-btn__icon` and the glyph column disappears automatically (`.px-btn:not(:has(> .px-btn__icon))`).
- Pick the glyph by meaning: `→` proceeds, `↗` opens or leaves, `↓` jumps down, `×` cancels or deletes, `↺` resets, `▸` runs.

**`.px-ibtn`**: a 48px square icon button. Always give it an `aria-label`, and usually a `data-tip`.

```html
<button type="button" class="px-ibtn" data-hatch="center" data-hover aria-label="Recentre" data-tip="Recentre on the survey"><span data-arrow="↺"></span></button>
```

**`.px-xbtn`**: an expand button. Only the square icon shows at rest; on hover or focus the panel and label wipe out to the right and corner ticks land. Structure: `__icon` (a permanently lit hatch), `__panel` (a hatch, with `__label` inside) and `__corners` (four spans). `.is-open` holds it open.

**Submit (`.px-xbtn--submit`)**: a 5u fixed part. The **stateful** variant uses `data-state="idle|busy|done|fail"` with matching `[data-show="…"]` children in both the icon and the label. Ink goes bone, then ash, then bright bone or signal. After changing the state, call `recolorHatches(button)`, set `aria-busy` and `aria-disabled`, and update the `.sr-only` name. See `Submit: normal + stateful` for the reference `setState`.

### 7.2 Text inputs

```html
<label class="px-input" data-validate="email" data-required data-error="Needs a name, an @ and a domain.">
  <span class="t-micro muted">Email</span>
  <span class="px-input__box">
    <input type="email" autocomplete="email" placeholder="surveyor@field.org" spellcheck="false">
    <span class="px-input__bar" data-hatch="x" data-hatch-cell="6" data-hatch-sweep="160" data-hatch-jitter="30"></span>
  </span>
  <span class="px-input__help t-micro">Checked when you leave the field.</span>
</label>
```

- **Parts:** `__prefix` (such as `https://`, or an icon with `--icon`), `__suffix`, `__kbd` (a shortcut hint), and `__btn` (in-box buttons such as reveal, clear and steppers, which are hatches with `data-hover`).
- **Focus effect** (global, chosen by `html[data-focus]` and saved under `localStorage['pixel-flow:focus-effect']`):
  - `bar`: the 6px `.px-input__bar` hatch sweeps in under the box in `--bone-700`.
  - `grid`: a breathing `.px-input__grid` of dim cells, added to every `.px-input__box` at load, fills the box.

  Both turn signal on error.
- **Validation:** `data-validate="email|url|text"` (add kinds to `VALID`), `data-required`, and `data-error`. Errors appear on blur or submit and clear as soon as the value is fixed. The field element gets `.validate()`, which returns a boolean. The error state is `.px-input.is-error`, `aria-invalid`, and the help text replaced; recolour with `recolorHatches(box)`.
- **Behaviours by attribute:** `data-password` (show and hide plus an 8-cell strength meter), `data-search` (a clear button), and `data-number` with `data-min`, `data-max` and `[data-step]` buttons (a stepper).
- Caret colour is signal; placeholders are ash-700.

### 7.3 Select, combobox and tag input

All three share `listbox(dd, field, pick)` (under the `Listbox` banner). It handles opening and closing, `aria-expanded`, `aria-activedescendant`, the highlight (`.is-on` on the option, which lights its hatch) and outside clicks; `listKeys()` provides the shared keyboard handling.

- `div.px-dd[data-select]`: a `button.px-input__box.px-select` with `.px-select__value` and `.px-dd__caret[data-arrow="↓"]`, followed by `ul.px-menu[role=listbox][data-hatch-group]` whose `li[role=option][data-hatch="x"]` items use `aria-selected`.
- `div.px-dd[data-combobox]`: an `input[role=combobox]` in a box that filters options; matches are wrapped in `<mark>` (signal), and `.px-menu__empty` shows when nothing matches.
- `div.px-dd[data-tags]`: `.px-chip` elements plus an input. Enter or comma adds a tag, Backspace removes the last, and an optional `li[data-add]` offers "Add new".

Menus open with a 4-step clip; the selected option shows a 6px signal square.

### 7.4 Choice controls

All of these are `button`s with real roles; `toggleAria()` flips `aria-checked` and dispatches **`px:change`** with `detail: boolean`. Wrap a control and its text in `label.opt` to make the text clickable.

```html
<label class="opt"><button class="px-switch" role="switch" aria-checked="true" data-hatch="x" data-hatch-cell="6" aria-label="Contours"><span class="px-switch__knob"></span></button><span>Contours</span></label>
<label class="opt"><button class="px-check" role="checkbox" aria-checked="false" data-hatch="scatter" data-hatch-cell="4" data-hatch-sweep="140" aria-label="Water"><svg viewBox="0 0 6 6" aria-hidden="true">…check pixels…</svg></button><span>Water</span></label>
```

- **Switch:** 48×24px with a signal hatch when on.
- **Check:** 24px with a bone scatter hatch and a pixel tick.
- **Radio:** `.px-radio` inside `[role=radiogroup]`, with a signal dot assembling from the centre (`data-hatch="center" data-hatch-cell="4"`). Arrow keys move along the group, wrapping.

### 7.5 Segmented controls

`.px-seg` is a row of buttons whose selection fills with an `entry` hatch that **leaves toward the new pick and enters from the old one's side**.

```html
<div class="px-seg px-seg--block" role="tablist" aria-label="View">
  <button role="tab" aria-selected="true" data-hatch="entry" data-hatch-cell="6" data-value="plan"><span>Plan</span></button>
  <button role="tab" aria-selected="false" data-hatch="entry" data-hatch-cell="6" data-value="relief"><span>Relief</span></button>
</div>
```

- Use `role="tablist"`/`tab` with `aria-selected` for view switches, or `role="radiogroup"`/`radio` with `aria-checked` for a pure choice of value.
- `--block` stretches the segments to full width.
- **2D grid:** add `.px-seg--grid` and `data-cols="3"`. Arrow keys then move across and down, without wrapping, and the fill sweeps vertically when you move up or down. The toast position picker (`#toast-pos`) is the reference, including the `.px-pos` screen glyph.
- Segments without an `id` are wired automatically. Give it an `id` and call `initSeg(seg, (value, el) => …)` to react; it returns `{ pick(value) }`.

### 7.6 Sliders

- **Stepped (`.px-slider`):** a track of 6px cells that light up to a signal head. Configure with `data-min`, `data-max`, `data-step`, `data-value` and `data-unit`. The existing wiring feeds the motion lab through `data-key`, so for a new slider add your own `input` listener.
- **Continuous (`.px-range`):** a solid bone fill with a signal thumb line over a dotted rail. Configure with `data-min`, `data-max`, `data-value`, `data-unit` and `data-decimals`; `--p` holds the 0–1 position.

Both use an invisible native `input[type=range]` laid over the track, so keyboard and screen-reader support are native. The track outline turns signal on focus.

### 7.7 Feedback

- **Progress (`.px-progress`):** two rows of cells filling in bone with a signal head. Add `role="progressbar"` and keep `aria-valuenow` updated. The demo's `Progress` object drives `#progress`.
- **Tags (`.px-tag`):** a 2×2 dot plus a label, with variants `--live` (signal, blinking), `--sync` (bone, blinking) and `--idle`.
- **Spinner (`.px-spin`):** a lit cell running round a 3×3 ring; `.px-dots` gives animated ellipses.
- **Toasts:** `toast(kicker, message, { kind, action, duration })` returns a handle `{ update(next), dismiss() }`.
  - **Kinds:** `info` (bone), `success` (bright bone with a pixel tick), `error` (a signal surface, `role="alert"`, stays until dismissed) and `progress` (breathes with a spinner until updated).
  - **Actions:** `action: { label, run(handle) }` adds a button.
  - **Timing and size:** hold time scales with text length (3.6–10s, or 6s or more with an action). Hovering pauses it. At most four stack, newest nearest the edge. Long text grows the toast by whole 24px lines, up to three.
  - **Position:** `toast.place('top-left' | 'top-center' | 'top-right' | 'bottom-left' | 'bottom-center' | 'bottom-right')` moves the stack (`#toasts[data-pos]`, saved under `pixel-flow:toast-position`). Toasts enter from the screen's interior and clear out toward their edge.
- **Dialogs** (native `<dialog>` with a modal focus trap):

```html
<button type="button" class="px-btn" data-hatch="x" data-hover data-dialog="#dlg-publish" aria-haspopup="dialog">…</button>

<dialog class="px-dialog" id="dlg-publish" aria-labelledby="dlg-publish-title" aria-describedby="dlg-publish-desc">
  <div class="px-dialog__scrim" data-hatch="scatter" data-hatch-cell="12" data-hatch-sweep="280"></div>
  <div class="px-dialog__panel" data-hatch="entry" data-hatch-cell="12" data-hatch-sweep="260" data-width="480">
    <div class="px-dialog__body">                     <!-- or a <form> for form dialogs -->
      <div class="px-dialog__head t-micro">
        <span><b>Publish</b> · 04 / Sector</span>
        <button type="button" class="px-ibtn px-dialog__close" data-hatch="center" data-hover data-dialog-close aria-label="Close"><span data-arrow="×"></span></button>
      </div>
      <div class="px-dialog__content">
        <h3 id="dlg-publish-title">Publish Ridgeline North?</h3>
        <p id="dlg-publish-desc">What will happen, in one or two sentences.</p>
      </div>
      <div class="px-dialog__foot">
        <button type="button" class="px-btn px-btn--quiet" data-hatch="x" data-hover data-dialog-close><span class="px-btn__label" data-roll>Cancel</span></button>
        <button type="button" class="px-btn px-btn--solid" data-hatch="x" data-hover data-dialog-close="publish" autofocus>…</button>
      </div>
    </div>
  </div>
</dialog>
```

  - **Motion:** the scrim's 12px cells dissolve in on the field grid, then the panel assembles from the side of its trigger and retracts toward it on close; the frame and content fade in last. The panel's width (`data-width`, whole units) and position snap to the field grid.
  - **API:** `Dialog.open(dlg, opener)` and `Dialog.close(dlg, value)`. The native `close` event fires after the exit animation, with the value in `dlg.returnValue`. Esc, the scrim and a bare `data-dialog-close` give `''`.
  - **Alerts:** `.px-dialog--alert` plus `role="alertdialog"` gives a signal kicker, ignores scrim clicks, and puts `autofocus` on the least destructive button.
  - **Focus:** focus goes to `[autofocus]` (an input's text is selected), page scroll is locked, and focus returns to the opener on close.
- **Tooltip:** add `data-tip="Text"` to any focusable element. A single shared bone `#px-tip` assembles above, or below near the top of the viewport. The first hover waits 400ms and neighbours then switch instantly; keyboard focus shows it immediately; Esc and scroll hide it. `aria-describedby` is managed for you.

### 7.8 Content

- **Links:** `.px-link` has a dotted pixel underline that a solid signal line wipes across in steps; `target="_blank"` adds `↗`. **Terms:** `.px-term` (dashed underline, `tabindex="0"`, usually with `data-tip`).
- **Card (`.px-card`, usually an `<a>`):** an `entry` hatch with `data-hatch-cell="12"`, `data-hatch-tone`, `data-hatch-breathe` and `data-hover`. It has `__top` (index and tag), an `h3` with a `p`, and `__foot` with `.bars[data-bars="2,3,…"]` (a mini 6-row bar chart that lights on hover) and a `.px-card__arrow`.
- **List (`.px-list` in a panel, with `data-hatch-group`):** 48px `.px-list__row` items with `data-hatch="diag" data-hatch-cell="12" data-hatch-tone=".2" data-hover`; `--head` is the 36px header row. On narrow screens, columns 4 and 5 hide.
- **Media (`figure.px-media`):** a greyscale image (colour on hover) under a `.px-media__veil` scatter hatch that breathes while `aria-busy="true"` and dissolves when the image is ready (remove `.is-on`). `.is-failed` turns the status signal. Wait for the veil to finish assembling before swapping images.
- **Skeleton:** `.px-skel` blocks are drawn as 5px cells with a stepped shimmer. Put the ghost and real layers in one `.skel` (`__ghost` and `__real` share a grid cell) and switch `data-state="loading|loaded"` along with `aria-busy`.
- **Data displays:** the dot matrix (`Dot matrix display`) and signal matrix are canvas demos built on the shared `tickers` loop; reuse `setupCanvas` for new canvas widgets.
- **`data-copy="#HEX"`** on a button copies the value and confirms with a toast.

---

## 8. Accessibility checklist

- Use native elements first (`button`, `a`, `input`, `dialog`), then ARIA roles that match the visual control (`switch`, `checkbox`, `radio`, `tab`, `option`, `combobox`, `listbox`, `progressbar`, `alertdialog`).
- The visuals must be driven by the ARIA state. If `aria-checked` doesn't change, neither should the hatch.
- Every icon-only control needs `aria-label`. Pixel SVGs are `aria-hidden="true"`.
- Focus is a 1px signal outline with an offset (3–4px outside, or −5px inside cramped boxes). Never remove it without a replacement that is at least as visible.
- `:focus-visible` also lights the hatch, so keyboard users see the same assembled state as hover.
- Body text must be bone, ash-300 or ash on black. Keep ash-700 for placeholders and meta.
- Overlays close on Esc and return focus to their trigger; status changes are announced (`role="status"`, `aria-live`, toast roles).
- Reduced motion is handled globally. If you add JS animation, gate it on `REDUCE`.

---

## 9. Recipes

### 9.1 A new page or screen

1. Start from the page skeleton (3.1). Keep `#field`, the header, `#toasts` and `#px-tip`.
2. Lay content out in `.wrap > .grid` with `.s-*` spans. Wrap groups in `.panel[data-panel][data-snap]` and text headers in `[data-quiet]`.
3. Use existing components with their exact markup; use only tokens for colour and units for size.
4. Pick one primary action per view (`--signal` or `--solid`). Everything else is default or `--quiet`.

### 9.2 A new component

1. **Size it in units:** a 24, 36 or 48px height, 1px `inset` box-shadow rules in `--void-500`, and padding in `--u`.
2. **Give state changes a hatch:** add `data-hatch` (choose the mode by meaning: `x` for a fill, `center` for a dot or icon, `scatter` for a toggle or veil, `entry` for selections and surfaces that relate to a point, `diag` for rows), set `--hatch`, and put the content above it with `z-index: 1`.
3. **Drive it from state:** use an ARIA attribute or class covered by `HATCH_ON`, or add a child-combinator entry there.
4. **Text over a lit hatch** switches to `--void` at weight 500, with a short delay (`transition-delay: 90ms`) so it lands with the cells.
5. **Make it accessible:** keyboard support, a role, an ARIA state, a visible focus style and an Esc path where relevant.
6. **If it's created or shown later,** call `initHatch()` on new hosts, `snapButtons(root)` after showing, and build hatches *before* switching them on (wait a frame) so they animate.
7. **Document it:** add a demo panel to the HTML and an entry here.

### 9.3 A new status or kind

Express it with existing ink:

| Status | Expression |
|---|---|
| Idle | Bone |
| Working | Ash, plus a spinner or breathing |
| Done | Bright bone, plus a tick glyph |
| Problem | Signal |

Change the meaning with the glyph and the label, never with a new colour.

---

## 10. Pitfalls (learned the hard way)

- **Hidden hosts don't build.** `buildHatch` skips zero-size elements (closed dialogs, `display: none`). Build on show, then add the on-state on the next frame, or the hatch starts fully on with no animation.
- **Hidden buttons measure 0.** `snapButtons` leaves them unsized; re-run it for the subtree once it's visible.
- **`.is-on` on a container flips every label inside** (descendant roll and arrow rules). Use specific state classes and child-combinator `HATCH_ON` entries.
- **`--cols` is taken** by the layout grid.
- **Thin SVG strokes on half pixels vanish.** Content inside `.wrap` often sits on .5px offsets, and the display may scale by 1.25 or 1.5. Draw pixel glyphs as filled shapes on whole user units, with `shape-rendering: crispEdges`, and give the `<svg>` `overflow: visible`.
- **The script is closure-scoped.** Calling `toast()` from the console or another script fails; add code inside the IIFE.
- **`buildHatch` resets entry and exit points** to the defaults when it rebuilds (on resize). Persist custom points in `data-hatch-entry` and `data-hatch-exit`.
- **Attribute changes outside `class`, `aria-checked` and `aria-selected` aren't observed;** call `queueHatchSync()` yourself.

---

## 11. Testing

- Serve over HTTP (`python -m http.server <port>` in `pixels/`); `file://` breaks fetches and some tooling. Add a cache-busting query (`?v=2`) after edits.
- Check every component with the mouse, the keyboard alone, `prefers-reduced-motion: reduce`, narrow widths (the 599 and 791px breakpoints) and a scaled display (125% and 150%).
- Hatch progress can be measured directly: `host._hatch.img.data` holds each cell's RGBA (alpha is the cell's value), and `host._hatch.on` and `.to` give the state.
- Automated browsers without OS focus won't match `:focus-visible` from synthetic events, and synthetic Esc doesn't trigger a dialog's `cancel` event. Dispatch `focusin` or `cancel` yourself in tests.

---

## 12. Code map

Search the HTML for these banners (CSS uses `/* ═══…═══ Name ═══…═══ */`; the script uses `/* ═════════════ Name ═════════════ */`).

| Area | CSS banner | Script banner or symbol |
|---|---|---|
| Tokens | `Tokens` | `U`, `REDUCE`, `FINE`, `$`, `$$`, `clamp`, `rnd` |
| Field | `Field` | `Field: page-wide pixel grid` (`Field.relayout`, `Field.setRunning`) |
| Layout and panels | `Layout`, `Panel` | `Snap surfaces to whole units` (`snapRO`, `snapButtons`) |
| Hatch engine | `Hatch` | `Hatch: per-cell delays…` (`HATCH_ON`, `initHatch`, `buildHatch`, `setHatchState`, `stagger`, `sweepFront`, `recolorHatches`, `PRESS_HOSTS`) |
| Roll and glyphs | inside `Hatch` (`.roll`, `.arrow`) | `Roll labels + arrows` |
| Buttons | `Button` (`.px-btn`, `.px-xbtn`, `.px-ibtn`, `.px-spin`) | `Submit: normal + stateful` |
| Inputs and pickers | `Controls` | `Fields: validation…` (`VALID`), `Listbox…` (`listbox`, `listKeys`), `Radio`, `Slider` |
| Switches and segments | `Controls` (`.px-switch`, `.px-check`, `.px-seg`) | `Switch · checkbox · segments` (`toggleAria`, `initSeg`) |
| Feedback | `Feedback` | `Progress`, `Toasts` (`toast`, `toast.place`) |
| Dialog | `Dialog` | `Dialog` (`Dialog.open`, `Dialog.close`) |
| Tooltip and links | `Text` | `Tooltip` |
| Card, list, media, skeleton | `Card`, `List`, `Image`, `Skeleton` | `Image`, `Skeleton` |
| Motion demos | `Motion` | `Motion lab`, `Looping demos` |
| Canvas widgets | `Matrix + Signal` | `Shared canvas ticker`, `Dot matrix display`, `Signal matrix` |

Persistent preferences: `localStorage['pixel-flow:focus-effect']` (`bar` or `grid`) and `localStorage['pixel-flow:toast-position']`.
