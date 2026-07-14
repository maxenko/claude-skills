# Interaction States — the full reference

Every interactive element must answer the user's hand. This file defines the complete state
vocabulary, how states combine, exact implementation techniques, and the timing that makes an
interface feel engineered rather than decorated.

## Contents

- The state matrix
- Implementation techniques (state layers, overlays, filled controls, hover-revealed)
- Timing and easing
- Focus — the keyboard's hover
- Cursors
- Hit areas
- Scrollbars
- Verification

## The state matrix

| State | Trigger | Visual treatment | Notes |
|---|---|---|---|
| Rest | default | token-defined base | Must already *look* interactive in context (affordance from shape/placement, not just hover) |
| Hover | pointer over | content-color layer at ~7–8%, or fill shift for filled controls | Mandatory for every clickable/draggable/editable element — on desktop, hover **is** the affordance |
| Focus-visible | keyboard focus | 2px ring, 3:1 contrast vs adjacent colors | Keyboard only (`:focus-visible`, not `:focus`) — no ring on mouse click |
| Pressed / active | mouse down | layer at ~10–12%, optionally `scale(0.98)` on buttons | Must appear **instantly** (≤50ms); this is the click "landing" |
| Selected | persistent choice | accent layer at ~10–14% + optional accent detail (left bar, checkmark) | Distinct from hover; survives pointer leaving |
| Selected + hover | both | selected layer +~5% | States stack; verify the stack is visible |
| Disabled | inert | whole element at ~38–40% opacity; no hover/press response | Don't just gray the text — kill the reactions too. Keep default cursor |
| Loading | async in flight | spinner/skeleton **replacing content, preserving dimensions** | A button keeps its width; a table shows skeleton rows in real row geometry |
| Dragging | active drag | layer at ~16%, elevation lift, `grabbing` cursor | Source shows a "lifted" copy or ghost; target shows an insertion indicator |
| Error | invalid state | danger token on border/underline + message; never color alone | Pair with icon or text for color-blind users |

Calibration source: Material 3's measured state-layer opacities (hover 8%, focus 10%, pressed
10%, drag 16%, disabled content 38%). These are good defaults for any design language; tune ±2%
to taste but keep them *system-wide constants*, never per-element.

## Implementation techniques

### 1. `color-mix` state layers (preferred — modern Chromium/WebView2/WebKit)

```css
.interactive:hover  { background: color-mix(in srgb, currentColor 7%, transparent); }
.interactive:active { background: color-mix(in srgb, currentColor 11%, transparent); }
```

Why `currentColor`: the layer derives from the element's *text* color, so one rule is correct on
every surface and in every theme — light text on dark surfaces produces a lightening layer, dark
text on light surfaces a darkening one. This is the whole trick.

### 2. Overlay pseudo-element (fallback, or when background is already used)

```css
.interactive { position: relative; }
.interactive::after {
  content: ""; position: absolute; inset: 0; border-radius: inherit;
  background: currentColor; opacity: 0; transition: opacity 120ms ease-out;
  pointer-events: none;
}
.interactive:hover::after  { opacity: 0.07; }
.interactive:active::after { opacity: 0.11; }
```

Composites on the GPU; never touches layout. Use when the element has a meaningful background of
its own (cards, filled inputs).

### 3. Filled controls (primary/danger buttons)

A state layer over a saturated fill reads as dirty. Instead pre-compute hover/active fill tokens:

```css
.btn-primary          { background: var(--accent); color: var(--on-accent); }
.btn-primary:hover    { background: var(--accent-hover); }   /* ≈ 8% toward white (dark) / black (light) */
.btn-primary:active   { background: var(--accent-active); }  /* ≈ 12% same direction */
```

### 4. Hover-revealed controls (progressive disclosure)

Row actions, close buttons on tabs, drag handles, checkbox columns — exist always, *show* on
hover, so the layout never shifts and the control is keyboard-reachable even when invisible:

```css
.row .row-actions { opacity: 0; transition: opacity 100ms ease-out; }
.row:hover .row-actions,
.row:focus-within .row-actions,
.row.selected .row-actions { opacity: 1; }
```

`focus-within` is not optional — a keyboard user tabbing into the hidden button must see it.

## Timing and easing

| Interaction | Duration | Easing | Why |
|---|---|---|---|
| Hover in | 100–150ms | ease-out | Fast acknowledgment; ease-out front-loads the change |
| Hover out | 150–200ms | ease-out | Slightly lazier exit reads as smooth, not jumpy |
| Press | 0–50ms | none/linear | The click must land NOW; any delay feels spongy |
| Release → rest | ~100ms | ease-out | |
| Expand/collapse, popover | 150–250ms | ease-out (enter), ease-in (exit) | Entering elements decelerate; exiting accelerate |
| Dialog/panel transitions | 200–300ms | cubic-bezier(0.2, 0, 0, 1) | Anything longer feels like waiting |
| Tooltip show delay | 400–600ms hover delay, then ~100ms fade | | Warm-up: once one tooltip shows, siblings show instantly while pointer keeps moving |

Rules that don't bend:
- Animate only `transform`, `opacity`, and color properties. Width/height/margin/top/left
  animations cause layout thrash and stutter under load.
- Name every transitioned property; never `transition: all` (it animates properties you didn't
  intend, including layout ones added later).
- `@media (prefers-reduced-motion: reduce)` → collapse durations to 0.01ms for movement
  (transform) animations; color/opacity fades may stay.
- One `transform` per element: if a component both scales on hover and translates while
  dragging, compose them in a single transform or split across parent/child nodes.

## Focus — the keyboard's hover

- Use `:focus-visible`, not `:focus`: ring on keyboard navigation only, never on mouse click.
- Ring spec (WCAG 2.4.7 / 2.4.13): at least 2px thick around the element's perimeter, ≥3:1
  contrast against both the element and its background. One `--focus-ring` token, used
  everywhere — a consistent ring is a navigation aid, a varied one is noise.
- `outline` + `outline-offset`, not `border` or `box-shadow`: outlines don't affect layout,
  respect `border-radius` in modern engines, and aren't clipped by `overflow: hidden` parents.
  Use negative offset (`outline-offset: -2px`) inside dense containers (table rows, list items)
  so rings don't collide.
- Every mouse path has a keyboard path: rows focusable and activatable with Enter, hover-revealed
  actions shown via `:focus-within`, Esc dismisses everything transient, arrows navigate within
  composite widgets (menu, tabs, tree, table), Tab moves between widgets — not through every
  row of a 500-row table (roving tabindex).

## Cursors

| Cursor | Use for | Never for |
|---|---|---|
| default (arrow) | buttons, rows, tabs, checkboxes — everything that *acts* | — |
| pointer (hand) | true links: navigation to elsewhere | buttons (dilutes the link signal; native apps never do it) |
| text | editable or selectable text | labels, headings that aren't selectable |
| grab / grabbing | draggable elements (grabbing while held) | — |
| col-resize / row-resize | split-pane dividers, table column edges | — |
| default (still!) | disabled controls | `not-allowed` reads as scolding; inertness says enough |

The corollary: since buttons don't get the hand cursor, the hover state carries the entire
"this is interactive" signal. That is why hover feedback is mandatory, not decorative.

## Hit areas

Visible size and hit area are separate decisions. Desktop targets can be small (16px icon
buttons), but the *hit area* should be ≥24×24px — pad the interactive element, or overlay an
invisible expanded hit zone (`::before` with negative inset). Split-pane dividers: 1px visible
line, 6–8px grab zone, with a hover highlight on the visible line so the affordance is
discoverable.

## Scrollbars

Default chunky scrollbars are the loudest "web page in a window" tell in a WebView-shipped
desktop app — and the thumb is an interactive element, so it gets hover/active states like
everything else.

- Prefer the CSS standard: `scrollbar-width: thin` plus
  `scrollbar-color: var(--border-strong) transparent`. Modern Chromium/WebView2 supports it,
  and when both are set it overrides the legacy pseudo-elements.
- For full control (radius, hover states) use `::-webkit-scrollbar` (~10px wide),
  `::-webkit-scrollbar-thumb` (token color, `border-radius: 5px`, a transparent border as
  inset padding) — and brighten the thumb on `:hover` and `:active`. Caveat: setting a webkit
  scrollbar width forces classic (non-overlay) scrollbars.
- Track stays transparent or `--surface-sunken`, never accent-colored — scrollbars are chrome,
  and chrome recedes.

## Verification

States are runtime behavior — verify them in the running app, not by reading the CSS. Drive
real hover/press/focus (project probe tooling, or manual), capture frames, and look. Pay
special attention to: selected+hover stacking, focus ring on dark surfaces, hover-revealed
controls via keyboard (`focus-within`), and disabled elements genuinely not reacting.
