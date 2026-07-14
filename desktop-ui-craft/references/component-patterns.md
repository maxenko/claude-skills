# Component Patterns — API design and completeness checklists

Two things make a component library feel senior: APIs that stay small as usage grows, and
components that handle their real states. This file covers both.

## Contents

- Component API design: the decision framework, state ownership, reuse discipline
- Completeness checklists: button, text input, select/dropdown, data table, dialog,
  menu/context menu, tabs, tooltip, toast, tree view, toolbar, split pane,
  empty/loading/error states

## Component API design

### The decision framework

| Variation kind | Mechanism | Example |
|---|---|---|
| Closed set of visual roles | `variant` prop | `variant: primary \| secondary \| ghost \| danger` |
| Closed set of dimensions | `size` prop | `size: sm \| md` (two sizes is usually enough) |
| Arbitrary content in a region | slot / children | card header, dialog footer, cell renderer |
| Multiple parts sharing state | compound components | `Tabs`+`Tab`, `Menu`+`MenuItem`, `Select`+`Option` |
| Behavior swap | separate component | a Tree is not a `List nested=true` |

Warning signs, and what they mean:
- A third content-shaped prop on the same region (`headerText`, `headerIcon`, `headerBadge`) →
  collapse into one `header` slot.
- A boolean prop that rearranges internal layout (`compact`, `horizontal`) → probably two
  components sharing extracted pieces, or a proper `layout` variant designed as such.
- A prop forwarded through two layers untouched → the intermediate component should take
  children/slots instead of re-exposing its child's API.
- Callers reaching into internals (styling descendant selectors, DOM poking) → the API is
  missing a variant or slot; add it rather than letting callers fork.

### State ownership

Support both controlled and uncontrolled use where a component has interactive state
(selection, expansion, open/closed): uncontrolled with an `initial*`/default for quick use,
controlled (`value` + `onChange`) when the parent must know. Never half-controlled (reading a
prop once into local state and ignoring updates) — that is where "why doesn't it update" bugs
live.

### Reuse discipline

- Extract on the second occurrence, not the fifth — visual consistency decays fastest through
  copy-paste-tweak.
- Extend the existing component (new variant, new slot) before writing a sibling. A near-
  duplicate (`CompactTable`, `UserCard2`) is a fork of the design system.
- Shared primitives compose upward: `Button` is used *inside* `Dialog` footers, `Toolbar`,
  `EmptyState` — never re-implemented there.
- Organization: one directory per component (implementation + styles + tests colocated);
  tokens and global utilities central; a deliberate public surface (what other modules may
  import) rather than everything-exported.

## Completeness checklists

A component ships when it passes its checklist — not when the happy path renders. These lists
are the difference between a component and a stub.

### Button
- Variants: primary (max one per view), secondary, ghost/tertiary, danger.
- Icon support: leading, trailing, icon-only — icon-only REQUIRES a tooltip.
- Loading: spinner replaces label, width preserved (no layout jump), input blocked.
- Full state matrix: hover, focus-visible, pressed (instant), disabled (no reactions).
- Enter/Space activate; disabled is skipped or announced, never a dead click.

### Text input
- Real label above the field — placeholder is a hint (tertiary contrast), never the label.
- Error state: border/underline in danger + message below + does not shift layout (reserve the
  message line or animate max-height); help text slot.
- Prefix/suffix slots (icons, units, clear button — clear shows on hover/focus when non-empty).
- Distinct disabled (inert, faded) vs read-only (selectable, no edit affordance).
- Focus ring on the field, `text` cursor, sensible width (not always 100%).

### Select / dropdown / combobox
- Keyboard: arrows navigate, typeahead jumps, Enter selects, Esc closes and restores.
- Selected item marked (checkmark), groups with headers, max-height + internal scroll.
- Popup at least trigger width; flips when near screen edge; closes on outside click.
- Empty/filtered-to-nothing state inside the popup.

### Data table (the flagship — business apps live here)
- Column headers: click to sort with asc/desc/none cycle and a visible indicator; hover state
  on sortable headers.
- Row hover (state-layer), row selection (checkbox column that appears on hover +
  focus-within + when any selection exists), selected style distinct from hover, both stack.
- Bulk-action bar appears only when a selection exists (count + actions).
- Row actions revealed on hover/focus-within, right-aligned; context menu on right-click
  duplicates them.
- Density toggle (compact ~28px / default ~34px / comfortable ~42px rows), persisted.
- Sticky header; column resize (col-resize cursor, 6–8px grab zone); truncation with
  ellipsis + title/tooltip on overflow.
- Numeric columns right-aligned with `font-variant-numeric: tabular-nums`; dates/statuses
  consistently formatted.
- Empty state (icon + one line + action), loading (skeleton rows in real geometry, not a
  spinner in a void), error (message + retry).
- Virtualize beyond a few hundred rows; keyboard: roving tabindex, arrows move row focus,
  Enter opens, Space toggles selection.
- Prefer hover-highlight over zebra striping; add zebra only when rows are tall/multi-line.

### Dialog / modal
- Scrim dims the app; Esc closes; outside click closes non-destructive dialogs only.
- Focus trapped inside; on close, focus returns to the trigger.
- Header: title + close icon-button (with hover state); footer: actions right-aligned as a
  group. Order follows the host platform — Windows puts the affirmative action left of Cancel,
  macOS puts it rightmost with Cancel adjacent. Pick the app's primary platform once and stay
  consistent in every dialog.
- Destructive confirms: danger-styled verb ("Delete 3 items", never "OK"); consider requiring
  typed confirmation for irreversible bulk operations.
- Sizes from a fixed set (sm/md/lg), body scrolls internally, never nest modals.

### Menu / context menu
- Hover highlight per item; keyboard arrows + Enter + Esc; type-to-jump.
- Icon column aligned (items without icons still align labels); shortcut hints right-aligned
  in tertiary text.
- Submenus: open on hover after ~200–300ms with a "safe triangle" for diagonal pointer travel,
  and on ArrowRight.
- Separators to group, sparingly; danger items styled; disabled items visible but inert.
- Opens at pointer (context) or anchored to trigger (dropdown); flips at screen edges.

### Tabs
- Active indicator (2px underline or pill); inactive tabs get hover; ArrowLeft/Right move,
  Home/End jump.
- Closeable tabs: close button appears on hover/active (with its own hover state);
  middle-click closes; unsaved-changes dot replaces close until hovered.
- Overflow: scroll with edge fades or an overflow menu — never wrap to a second row.

### Tooltip
- 400–600ms show delay; once one is shown, siblings show instantly (shared warm-up timer).
- Never contains interactive content (that's a popover); never blocks the pointer target.
- Icon-only buttons always have one; include the keyboard shortcut when one exists.
- Also shows on keyboard focus, not just hover.

### Toast / notification
- One corner, stacked, max ~3 visible; auto-dismiss 4–6s, paused on hover.
- One optional action link (Undo); errors persist until dismissed.
- Never use for information the user must act on — that's a dialog or inline message.

### Tree view
- Chevron rotates on expand; indent guides for depth; ArrowLeft/Right collapse/expand,
  Up/Down move, type-to-jump.
- Selection distinct from focus; hover state on rows; inline rename on F2/double-click.
- Lazy children show an in-row loading affordance.

### Toolbar
- Icon buttons with tooltips (+shortcuts); related controls grouped with separators between
  groups only.
- Toggle buttons show a pressed/active look distinct from hover; overflow menu when the
  window narrows (priority order, least-used collapse first).

### Split pane / resizable panels
- 1px visible divider, 6–8px invisible grab zone, divider highlights on hover,
  col/row-resize cursor.
- Min sizes enforced; double-click divider resets to default; sizes persist across sessions.
- Drag uses a full-viewport glass overlay so fast drags don't drop; pane content handles its
  own overflow at min size.

### Empty / loading / error states (every data surface has all three)
- Empty: icon or subtle illustration + one explanatory line + the action that fills it
  ("No patterns yet — create one"). Never a blank void, never a wall of text.
- Loading: skeletons that match real content geometry for structured content; spinners only
  for small inline waits; show nothing for waits under ~200ms (flash avoidance).
- Error: what failed in plain words + a retry affordance; keep stale data visible with a
  "couldn't refresh" notice rather than blanking it.
