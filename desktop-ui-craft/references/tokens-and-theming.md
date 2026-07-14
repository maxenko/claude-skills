# Tokens & Theming — architecture and starter set

Design tokens are the mechanism behind "consistent AND customizable." Get the architecture
right and a theme change is one file; get it wrong and every restyle is a codebase-wide grep.

## Contents

- Three tiers, one reference direction
- Starter token set for a business desktop app
- Dark theme — derivation rules, not inversion
- Contrast requirements (verify, don't eyeball)
- Mechanics that bite (reactivity, WebView var() quirks, per-theme completeness)

## Three tiers, one reference direction

```
component tokens  →  semantic tokens  →  primitive tokens
--btn-primary-bg      --accent             --blue-600
(optional tier)       (main consumption)   (raw palette)
```

- **Primitives** (`--gray-100…900`, `--blue-400`, `--space-4`): raw values, no meaning. The
  complete palette. Stable — changes only on rebrand.
- **Semantic** (`--surface`, `--text-secondary`, `--border-subtle`, `--accent`): purpose-named
  aliases into primitives. **This is the tier components consume.** Theming = the same semantic
  names pointing at different primitives per theme.
- **Component** (`--table-row-hover`, `--btn-radius`): only when a component genuinely needs
  independent overriding. Most components never need this tier — don't manufacture it.

Never skip the direction: a component pointing straight at `--gray-300` (or a raw hex) has
opted out of theming. Name semantic tokens by *purpose*, never appearance — `--text-muted`, not
`--light-gray` (which becomes a lie in dark mode).

**Naming grammar** for growth without chaos: `[concept]-[variant]-[state]` —
`--surface`, `--surface-raised`, `--text-primary`, `--border-strong`, `--accent-hover`,
`--danger-bg-subtle`. Consistent grammar means a developer can *guess* a token name correctly.

## Starter token set for a business desktop app

This is a complete, sufficient set — resist adding more until a real surface has no home.

```css
:root {
  /* Surfaces — a ladder, darkest context to most elevated */
  --bg: #f6f7f9;               /* app frame / recessed areas */
  --surface: #ffffff;           /* primary work surface */
  --surface-raised: #ffffff;    /* cards, popovers (light mode differentiates by shadow) */
  --surface-sunken: #eef0f3;    /* wells, input backgrounds, code blocks */
  --overlay-scrim: rgb(15 18 22 / 0.45);

  /* Text — three contrast tiers + disabled */
  --text-primary: #1a1d21;      /* full contrast, ≥7:1 on --surface */
  --text-secondary: #55595f;    /* labels, metadata, ~70% emphasis, still ≥4.5:1 */
  --text-tertiary: #8b9096;     /* hints, placeholders, timestamps — large/incidental only */
  --text-disabled: #a9adb3;
  --text-on-accent: #ffffff;

  /* Borders */
  --border-subtle: #e4e7eb;     /* hairlines, table row separators */
  --border: #d2d6db;            /* inputs, cards */
  --border-strong: #9aa0a8;     /* emphasis, hover on inputs */

  /* Accent — exactly one, plus interaction variants */
  --accent: #3861d0;
  --accent-hover: #3156bd;      /* ~8% toward black (light theme) */
  --accent-active: #2b4daa;     /* ~12% */
  --accent-subtle-bg: rgb(56 97 208 / 0.10);  /* selection fills, active tab bg */
  --focus-ring: #3861d0;

  /* Status — each with a full-strength and a subtle-bg form */
  --danger: #c93a3a;  --danger-bg-subtle: rgb(201 58 58 / 0.10);
  --warning: #b57a1e; --warning-bg-subtle: rgb(181 122 30 / 0.12);
  --success: #2e8b57; --success-bg-subtle: rgb(46 139 87 / 0.10);

  /* Geometry */
  --radius-sm: 4px; --radius-md: 6px; --radius-lg: 8px;
  --space-1: 4px; --space-2: 8px; --space-3: 12px; --space-4: 16px;
  --space-5: 24px; --space-6: 32px; --space-7: 48px;

  /* Type — 4 sizes is usually enough for business UI */
  --font-ui: system-ui, "Segoe UI", sans-serif;
  --font-mono: ui-monospace, Consolas, monospace;
  --text-xs: 11px; --text-sm: 12.5px; --text-md: 13.5px; --text-lg: 16px;

  /* Elevation (light mode: shadows; dark mode: surface lightness instead) */
  --shadow-raised: 0 1px 3px rgb(15 18 22 / 0.10);
  --shadow-overlay: 0 4px 16px rgb(15 18 22 / 0.18);

  /* Motion */
  --dur-fast: 120ms; --dur-med: 200ms;
  --ease-out: cubic-bezier(0.2, 0, 0, 1);
}
```

Notes on the choices:
- Neutrals carry a slight cool/warm *cast* (not pure gray) — pick one temperature and keep the
  whole ladder on it. Warm-tinted gray reads calmer; blue-tinted reads crisper.
- Body UI text at 12.5–13.5px is normal desktop density (the web's 16px default is for prose).
- The spacing scale is the 4px grid; if a mockup needs 10px, the mockup is wrong.

## Dark theme — derivation rules, not inversion

A dark theme is a *re-mapping* of semantic tokens, produced by rules:

1. **Base is dark gray, not black** (`#151719`-ish). Pure black makes elevation impossible and
   maximizes eye-strain contrast with text.
2. **Elevation = lightness.** Shadows are invisible on dark; a raised surface is a *lighter*
   dark (`--surface-raised` a step above `--surface`, overlays another step). Zero out or
   drastically soften the shadow tokens.
3. **Desaturate accents 20–30%** and lighten them; saturated colors vibrate on dark backgrounds.
   Status colors too.
4. **Text is off-white** (`#e8eaed`-ish, ~90% white), never `#fff` — pure white on dark glows.
   The three contrast tiers persist (≈90% / 65% / 45% lightness).
5. **Borders lighten with elevation** and generally get *more* subtle — on dark UIs, surface
   lightness does the separating that borders do on light.

```css
[data-theme="dark"] {
  --bg: #101214; --surface: #16191c; --surface-raised: #1d2125; --surface-sunken: #0c0e10;
  --text-primary: #e8eaed; --text-secondary: #a3a9b0; --text-tertiary: #6e747c;
  --border-subtle: #24282d; --border: #2e3338; --border-strong: #4a5058;
  --accent: #6b8ee6;  /* lighter + ~25% desaturated vs light theme */
  --accent-hover: #7d9cea; --accent-active: #8fabee;  /* dark theme: hover goes LIGHTER */
  --shadow-raised: none; --shadow-overlay: 0 4px 20px rgb(0 0 0 / 0.5);
}
```

Note `--accent-hover` reverses direction: light themes darken on hover, dark themes lighten.
Encoding this in tokens (rather than a CSS `filter` or hand math in components) is exactly why
the interaction variants exist as tokens.

## Contrast requirements (verify, don't eyeball)

- Body/normal text: ≥4.5:1 against its actual surface (WCAG AA) — **in every theme**.
- Large text (≥18.7px bold / 24px): ≥3:1.
- UI component boundaries that carry meaning (input borders, icons, focus ring): ≥3:1.
- `--text-tertiary` frequently fails 4.5:1 by design — restrict it to incidental content that
  is duplicated or non-essential, never to values the user must read.
- Check the *subtle* backgrounds: text on `--accent-subtle-bg` and `--danger-bg-subtle` must
  still clear AA against the mixed result.

## Mechanics that bite

- Put the theme's token block on a root element whose re-render actually updates it in your
  framework; verify a live theme switch repaints everything (stale surfaces = a token defined
  in a non-reactive location).
- WebView quirk (Dioxus/Tauri/anything Chromium-embedded): a `var()` inside an **inline style
  that is re-patched between renders** can fail to re-resolve — the element paints nothing. For
  colors that toggle at runtime in inline styles, resolve the token to a concrete value in code
  and interpolate the literal; keep `var()` for stylesheet rules and never-changing inline uses.
- SVG: `var()` does not resolve in presentation attributes in some engines — set paint via
  `style="fill:var(--x)"`, keep geometry as attributes.
- Every new token gets a value in *every* theme at the moment of creation. A token defined only
  in `:root` silently falls back in dark mode and hides the bug until someone switches themes.
