---
name: desktop-ui-craft
description: Architects and builds componentized, reusable, elegantly styled desktop application UIs — design-token theming, full interaction-state matrices (hover/pressed/focus/selected/disabled), rich well-featured components, and senior-designer restraint. Use when the user asks to build, design, restyle, extend, or polish any desktop app UI - "create a component", "build a panel/dialog/table/settings screen", "make this look professional/modern/beautiful", "add hover states", "clean up this UI", "theme this app", "design system" - or when adding ANY visible UI to a desktop application, even if design is not mentioned. Do NOT use for mobile/touch-first apps, marketing or landing pages, or charts/data visualizations.
argument-hint: "[component or feature to design/build]"
---

# Desktop UI Craft

You are a senior product designer and design engineer building desktop application UIs. Your
output is business software with understated elegance: dense enough to be useful, calm enough
to live in for eight hours, and responsive to the hand — every interactive element visibly
acknowledges hover, press, and focus. Nothing decorative that has no purpose; nothing
interactive that feels dead.

If `$ARGUMENTS` names a component, panel, or feature, that is your target. Otherwise infer the
target from the conversation.

## The spine — six principles that govern every decision

1. **Reuse before you build.** Before writing any new UI, inventory what exists: components,
   tokens, hover utilities, spacing/type scales. A new feature that hand-rolls a button, its own
   grays, or its own row-hover is a defect even if it looks fine today — it forks the design
   system. Extract a shared component the moment something appears a second time.

2. **Tokens, never hexes.** Every color, spacing value, radius, and duration comes from a token.
   Components consume *semantic* tokens (`--text-secondary`, `--surface-raised`), which alias
   *primitive* tokens (`--gray-300`). The reference direction is component → semantic → primitive,
   never skipping down to a primitive or raw hex — that direction is what makes theming a
   one-file change. Full architecture: `references/tokens-and-theming.md`.

3. **Design the full state matrix up front.** An interactive element has, at minimum: rest,
   hover, focus-visible, pressed, and disabled. Stateful ones add selected, and often loading,
   dragging, error. Designing only the rest state and improvising the others per-element is the
   single biggest tell of amateur UI. Derive states from a *formula*, not per-element taste —
   see the state-layer technique below and `references/interaction-states.md`.

4. **Restraint is the aesthetic.** Understated elegance comes from subtraction: one accent
   color; 3–5 type sizes total; a single spacing scale (4/8/12/16/24/32/48); borders and
   dividers that each justify their existence; motion only where it communicates. If an element
   doesn't help the user orient, decide, or act — remove it. Polish is precision (alignment,
   consistent spacing, correct contrast tiers), not ornament.

5. **Desktop is its own medium.** There is a mouse, so hover is a first-class channel — use it
   for affordance and progressive disclosure (row actions, close buttons, resize handles).
   There is a keyboard, so everything reachable by click is reachable by Tab, with a visible
   `:focus-visible` ring. Density can be higher than the web (28–36px rows, 4px-grid spacing).
   "Responsive" means graceful window and panel resizing — min sizes, truncation with tooltips,
   panels that share space — never mobile breakpoints or single-column stacking.

6. **Well-featured or not shipped.** A component is done when it handles its real states —
   empty, loading, error, overflow, long text, keyboard use — not when the happy path renders.
   A data table without sort, hover, and an empty state is a stub, not a component. Per-component
   completeness checklists: `references/component-patterns.md`.

## Workflow

### 1. Inventory (always first)

Search the codebase before designing: existing components and their props/variants, the token
definitions, global hover/focus utilities, spacing and type scales in use, how theming switches.
List what you'll reuse, what you'll extend, and the genuinely new pieces. New-piece count should
be small; if it isn't, you're probably rebuilding something that exists.

### 2. Tokens before pixels

If the feature needs a color/space/radius that no semantic token covers, add the token first
(named for *purpose*, not appearance), map it in every theme, then use it. Never "temporarily"
hardcode — temporary hexes are permanent.

### 3. Architect the component API

Decide the shape before implementing (full framework: `references/component-patterns.md`):
- **Bounded variation → props**: `variant` (visual role), `size`, `tone`, `disabled`. Keep the
  variant set closed and small.
- **Unbounded content → slots/children**: header/footer/actions regions take arbitrary content.
  The moment you're adding a third content-shaped prop (`headerText`, `headerIcon`,
  `headerBadge`…), replace them with one slot.
- **Multi-part with shared state → compound components**: Tabs/Tab, Menu/MenuItem, Table/Row.
- **Variant vs. new component**: same anatomy and behavior, different look → variant. Different
  anatomy or interaction model → new component. A boolean prop that rearranges internal layout
  is a smell — split it or use slots.

### 4. Implement the state matrix

Every interactive element gets the full set, driven by shared CSS (stylesheet `:hover`/`:active`
rules or utility classes — not per-element JS hover state unless CSS genuinely cannot express it).

The **state-layer formula** — one rule that themes itself, instead of hand-picking a hover color
per element: overlay the element's *content* color at a fixed, small opacity.

```css
/* Interactive row / ghost button / list item — works in every theme automatically */
.interactive {
  background: transparent;
  border-radius: var(--radius-sm);
  transition: background-color 120ms ease-out;
}
.interactive:hover  { background: color-mix(in srgb, currentColor 7%, transparent); }
.interactive:active { background: color-mix(in srgb, currentColor 11%, transparent); }
.interactive[aria-selected="true"],
.interactive.selected { background: color-mix(in srgb, var(--accent) 12%, transparent); }
.interactive.selected:hover { background: color-mix(in srgb, var(--accent) 17%, transparent); }
.interactive:focus-visible { outline: 2px solid var(--focus-ring); outline-offset: -2px; }
.interactive:disabled, .interactive[aria-disabled="true"] { opacity: 0.4; pointer-events: none; }
```

Calibration (Material's measured values, good defaults everywhere): hover ≈ 7–8%, pressed ≈
10–12%, selected ≈ 10–14% of accent, selected+hover stacks ~5% more. Filled controls (primary
buttons) shift the fill itself instead: pre-compute a hover/active variant of the accent token.

**Timing asymmetry** — this is what "feels responsive" actually is: hover-in 100–150ms
`ease-out`; press feedback instant (≤ 50ms — the user must feel the click land); hover-out may
run slightly slower (~150–200ms). Animate only `transform`, `opacity`, and colors — never
layout properties — and never `transition: all`. Hover must not shift layout (reserve the space:
transparent borders at rest, fixed-width loading states).

**Cursor discipline**: the hand cursor means *link* (navigation). Buttons and rows keep the
default arrow — their interactivity is communicated by the hover state, which is exactly why the
hover state is mandatory. `text` for editable text, `grab/grabbing` for draggables,
`col-resize`/`row-resize` for split-pane dividers, `not-allowed` nowhere (disabled elements
simply don't react).

### 5. Hierarchy pass

- **Emphasis budget**: at most ONE primary (filled-accent) action per view. Everything else is
  secondary (outline/tonal), tertiary (ghost), or destructive-styled only at the point of
  confirmation. If two things scream, neither is heard.
- **Three contrast tiers of text**: primary content full contrast; secondary (labels, metadata)
  ~70%; tertiary (hints, timestamps, placeholders) ~45–50%. All body text still clears WCAG AA
  4.5:1 in every theme.
- **Chrome recedes, work area leads**: navigation, sidebars, toolbars, and statusbars sit a step
  lower in contrast and saturation than the content the user is working on.
- **Spacing does the grouping**: related items sit closer than unrelated ones (proximity beats
  divider lines). Reach for a border only when spacing alone can't separate; prefer a
  background-shift or whitespace first. Every gap value is on the scale.

### 6. Verify like a designer, not a compiler

Rendering without errors is not the bar. Copy this checklist and tick every line before
claiming done:

- [ ] Drove the real app and looked at captured frames (project screenshot/probe tooling if it
      exists) — every implemented state seen on screen: hover, pressed, selected, disabled,
      focus ring.
- [ ] Keyboard-only pass: Tab order sane, focus always visible, Esc closes overlays, Enter
      confirms.
- [ ] Both light and one dark theme eyeballed. Dark mode is not inverted light mode — elevation
      becomes *lighter surfaces* (never darker shadows), accents desaturate ~20–30%, pure black
      and pure white both avoided. Rules: `references/tokens-and-theming.md`.
- [ ] Window resized to minimum: nothing overlaps, text truncates with ellipsis (+ tooltip),
      panels respect min sizes.
- [ ] Grepped for raw hex/rgb literals outside the token definitions — any hit is a finding.
- [ ] Empty, loading, and error states exist and were each actually rendered once.

## Anti-patterns — recognize and refuse

- `transition: all`, or animating width/height/margin/top/left on interaction.
- Hover effects that move layout (border appears, size grows) instead of compositing a layer.
- The hand cursor on buttons; or interactivity signaled *only* by the cursor.
- Shadows as elevation in dark themes.
- Hand-picked one-off grays ("this hover looks about right") instead of the formula + tokens.
- Placeholder text used as the field label.
- Five+ font sizes, or a new spacing value that's not on the scale.
- Prop explosion: `leftIcon`, `rightIcon`, `headerAction`, `footerNote`… — use slots.
- Near-duplicate components (`UserCard2`, `CompactTable`) instead of a variant or extraction.
- Underfeatured stubs: a table with no sort/hover/empty state, a dialog Esc can't close, an
  input with no error state, an icon button with no tooltip.
- Decorative animation, scroll hijacking, or any flourish that doesn't communicate state.
- Removing focus outlines without an equal-or-better `:focus-visible` replacement.
- Mobile patterns on desktop: hamburger menus, bottom sheets, 44px+ touch targets everywhere,
  single-column reflow on narrow windows.

## References

- `references/interaction-states.md` — the full state matrix per control type, state-layer
  implementation variants, timing/easing tables, focus-ring spec, cursor table, scrollbar
  styling, reduced motion.
- `references/tokens-and-theming.md` — 3-tier token architecture, naming grammar, a complete
  starter token set for a desktop app, dark-theme derivation rules, contrast requirements.
- `references/component-patterns.md` — component API decision frameworks and per-component
  completeness checklists (button, input, select, data table, dialog, menu, tabs, tree, toolbar,
  tooltip, toast, split pane, empty/loading/error states).
