---
title: "CSS @layer Architecture"
categories:
  - css
  - architecture
  - design-system
tags:
  - css-layers
  - oklch
  - nesting
  - has-selector
  - container-queries
  - logical-properties
description: >-
  Organizing CSS with @layer for cascade control, OKLCH colors for perceptual uniformity,
  native CSS nesting, :has() for parent selection, container queries for component-level
  responsiveness, and logical properties. The 37signals approach to minimal utility classes.
---

# CSS @layer Architecture

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Layer Order and Responsibilities](#layer-order-and-responsibilities)
  - [OKLCH Color System](#oklch-color-system)
  - [Native CSS Nesting](#native-css-nesting)
  - [:has() Pseudo-Class for Parent Selection](#has-pseudo-class-for-parent-selection)
  - [Container Queries for Component Responsiveness](#container-queries-for-component-responsiveness)
  - [Logical Properties](#logical-properties)
  - [Minimal Utility Classes](#minimal-utility-classes)
- [Full Layer Architecture Example](#full-layer-architecture-example)
- [Pattern Card](#pattern-card)

## Overview

CSS `@layer` gives explicit control over the cascade, eliminating specificity wars and making overrides predictable. Instead of fighting `!important` or artificially inflating selectors, you declare the order of precedence once and every rule respects it.

The 37signals approach (used in Basecamp, HEY, and Campfire) combines `@layer` with a small set of modern CSS features: OKLCH colors for perceptual uniformity, native nesting to eliminate preprocessor dependencies, `:has()` for parent-aware styling, container queries for component-scoped responsiveness, and logical properties for internationalization-ready layouts.

The result is a CSS architecture that is powerful, readable, and maintainable with roughly 60 utility classes instead of Tailwind's thousands. The key insight is that semantic component styles with good cascade control need far fewer overrides than utility-first approaches.

## Implementation

### Layer Order and Responsibilities

Declare the layer order at the top of your main stylesheet. Layers declared later have higher priority in the cascade.

```css
/* app/assets/stylesheets/application.css */
@layer reset, base, components, modules, utilities;

@import "reset.css" layer(reset);
@import "base.css" layer(base);
@import "components.css" layer(components);
@import "modules.css" layer(modules);
@import "utilities.css" layer(utilities);
```

Each layer has a clear responsibility:

| Layer | Priority | Responsibility | Examples |
|-------|----------|----------------|----------|
| `reset` | Lowest | Normalize browser defaults | Box-sizing, margin removal, font smoothing |
| `base` | Low | Design tokens and element defaults | Custom properties, typography, link styles |
| `components` | Medium | Reusable UI patterns | Buttons, cards, badges, form controls |
| `modules` | High | Page-specific or feature-specific layouts | Dashboard grid, settings panel, onboarding flow |
| `utilities` | Highest | Single-purpose overrides | Spacing, visibility, text alignment |

```css
/* app/assets/stylesheets/reset.css */
@layer reset {
  *, *::before, *::after {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  html {
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-rendering: optimizeLegibility;
  }

  img, picture, video, canvas, svg {
    display: block;
    max-inline-size: 100%;
  }

  input, button, textarea, select {
    font: inherit;
    color: inherit;
  }

  body {
    min-block-size: 100dvh;
    line-height: 1.5;
  }
}
```

```css
/* app/assets/stylesheets/base.css */
@layer base {
  :root {
    /* Color tokens in OKLCH */
    --color-primary: oklch(55% 0.25 260);
    --color-primary-hover: color-mix(in oklch, var(--color-primary), white 15%);
    --color-primary-active: color-mix(in oklch, var(--color-primary), black 10%);

    --color-success: oklch(65% 0.2 145);
    --color-warning: oklch(75% 0.18 80);
    --color-danger: oklch(55% 0.22 25);

    --color-text: oklch(25% 0.02 260);
    --color-text-muted: oklch(55% 0.02 260);
    --color-surface: oklch(99% 0.005 260);
    --color-border: oklch(88% 0.01 260);

    /* Spacing scale */
    --space-xs: 0.25rem;
    --space-sm: 0.5rem;
    --space-md: 1rem;
    --space-lg: 1.5rem;
    --space-xl: 2rem;
    --space-2xl: 3rem;

    /* Typography */
    --font-sans: system-ui, -apple-system, sans-serif;
    --font-mono: ui-monospace, "Cascadia Code", "Fira Code", monospace;

    --text-sm: 0.875rem;
    --text-base: 1rem;
    --text-lg: 1.125rem;
    --text-xl: 1.25rem;
    --text-2xl: 1.5rem;

    /* Radius */
    --radius-sm: 0.25rem;
    --radius-md: 0.5rem;
    --radius-lg: 0.75rem;

    /* Shadows */
    --shadow-sm: 0 1px 2px oklch(0% 0 0 / 0.05);
    --shadow-md: 0 4px 6px oklch(0% 0 0 / 0.07);
    --shadow-lg: 0 10px 15px oklch(0% 0 0 / 0.1);
  }

  body {
    font-family: var(--font-sans);
    font-size: var(--text-base);
    color: var(--color-text);
    background-color: var(--color-surface);
  }

  a {
    color: var(--color-primary);
    text-decoration-thickness: 1px;
    text-underline-offset: 0.15em;

    &:hover {
      color: var(--color-primary-hover);
    }
  }

  h1, h2, h3, h4, h5, h6 {
    line-height: 1.2;
    text-wrap: balance;
  }
}
```

### OKLCH Color System

OKLCH (Oklab Lightness, Chroma, Hue) provides perceptually uniform colors. Unlike HSL, equal lightness values in OKLCH actually look equally bright. This makes it ideal for design systems where you need consistent contrast across color families.

```css
/* OKLCH format: oklch(lightness chroma hue) */
/* Lightness: 0% (black) to 100% (white) */
/* Chroma: 0 (gray) to ~0.4 (most vivid) */
/* Hue: 0-360 degrees on the color wheel */

:root {
  /* Primary scale - consistent lightness progression */
  --primary-50:  oklch(97% 0.02 260);
  --primary-100: oklch(93% 0.05 260);
  --primary-200: oklch(85% 0.10 260);
  --primary-300: oklch(75% 0.15 260);
  --primary-400: oklch(65% 0.20 260);
  --primary-500: oklch(55% 0.25 260);  /* Base */
  --primary-600: oklch(48% 0.22 260);
  --primary-700: oklch(40% 0.18 260);
  --primary-800: oklch(32% 0.14 260);
  --primary-900: oklch(25% 0.10 260);
}
```

Use `color-mix()` to derive hover and active states without defining every variant:

```css
.btn-primary {
  background-color: var(--color-primary);
  color: white;

  &:hover {
    /* Mix 15% white into the primary color for a lighter hover */
    background-color: color-mix(in oklch, var(--color-primary), white 15%);
  }

  &:active {
    /* Mix 10% black into the primary color for a darker press */
    background-color: color-mix(in oklch, var(--color-primary), black 10%);
  }

  &:disabled {
    /* Mix 50% of surface color to desaturate */
    background-color: color-mix(in oklch, var(--color-primary), var(--color-surface) 50%);
  }
}
```

### Native CSS Nesting

Native CSS nesting eliminates the need for Sass or other preprocessors. The `&` selector references the parent, just like in Sass, but runs natively in the browser.

```css
@layer components {
  .card {
    background: var(--color-surface);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    padding: var(--space-lg);
    box-shadow: var(--shadow-sm);

    & .card-header {
      display: flex;
      align-items: center;
      justify-content: space-between;
      margin-block-end: var(--space-md);

      & h3 {
        font-size: var(--text-lg);
        font-weight: 600;
      }
    }

    & .card-body {
      color: var(--color-text-muted);
      line-height: 1.6;
    }

    & .card-footer {
      margin-block-start: var(--space-lg);
      padding-block-start: var(--space-md);
      border-block-start: 1px solid var(--color-border);
      display: flex;
      gap: var(--space-sm);
      justify-content: flex-end;
    }

    /* State variants using nesting */
    &.card--highlighted {
      border-color: var(--color-primary);
      box-shadow: 0 0 0 1px var(--color-primary);
    }

    /* Hover state */
    &:hover {
      box-shadow: var(--shadow-md);
    }

    /* Responsive behavior inside the component */
    @media (width < 640px) {
      padding: var(--space-md);

      & .card-footer {
        flex-direction: column;
      }
    }
  }
}
```

### :has() Pseudo-Class for Parent Selection

The `:has()` pseudo-class selects an element based on its descendants or siblings. This enables parent-aware styling without JavaScript or extra CSS classes.

```css
@layer components {
  /* Style a form group differently when its input is focused */
  .form-group {
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    padding: var(--space-sm) var(--space-md);
    transition: border-color 0.15s;

    &:has(input:focus) {
      border-color: var(--color-primary);
      box-shadow: 0 0 0 2px color-mix(in oklch, var(--color-primary), transparent 80%);
    }

    &:has(input:invalid:not(:placeholder-shown)) {
      border-color: var(--color-danger);
    }

    &:has(input:disabled) {
      opacity: 0.5;
      pointer-events: none;
    }
  }

  /* Show a count badge only when the list has items */
  .nav-item {
    &:has(.badge:not(:empty))::after {
      content: "";
      display: block;
      inline-size: 6px;
      block-size: 6px;
      border-radius: 50%;
      background: var(--color-danger);
    }
  }

  /* Auto-hide empty containers */
  .flash-container:not(:has(.flash)) {
    display: none;
  }

  /* Style a table row based on a checkbox inside it */
  tr:has(input[type="checkbox"]:checked) {
    background-color: color-mix(in oklch, var(--color-primary), transparent 92%);
  }
}
```

### Container Queries for Component Responsiveness

Container queries let components respond to their own size, not the viewport. This makes components truly portable across different layouts.

```css
@layer components {
  .dashboard-card {
    container-type: inline-size;
    container-name: dashboard-card;
  }

  .dashboard-card-content {
    display: grid;
    gap: var(--space-md);
    grid-template-columns: 1fr;

    /* When the card container is wide enough, use two columns */
    @container dashboard-card (inline-size > 400px) {
      grid-template-columns: 1fr 1fr;
    }

    /* Three columns in large containers */
    @container dashboard-card (inline-size > 700px) {
      grid-template-columns: 1fr 1fr 1fr;
    }
  }

  /* Stat card adapts to its container, not the viewport */
  .stat-card {
    container-type: inline-size;
    container-name: stat;
    display: flex;
    align-items: center;
    gap: var(--space-md);
    padding: var(--space-md);

    & .stat-value {
      font-size: var(--text-2xl);
      font-weight: 700;
    }

    & .stat-label {
      font-size: var(--text-sm);
      color: var(--color-text-muted);
    }

    /* Stack vertically in narrow containers */
    @container stat (inline-size < 200px) {
      flex-direction: column;
      text-align: center;

      & .stat-value {
        font-size: var(--text-xl);
      }
    }
  }
}
```

### Logical Properties

Logical properties replace physical direction properties (`left`, `right`, `top`, `bottom`) with flow-relative equivalents. This makes layouts work correctly in both LTR and RTL writing modes without any changes.

```css
@layer components {
  .sidebar {
    /* GOOD: Logical properties work in both LTR and RTL */
    padding-inline: var(--space-lg);
    padding-block: var(--space-md);
    border-inline-end: 1px solid var(--color-border);
    margin-block-end: var(--space-xl);
    inline-size: 280px;
    max-block-size: 100dvh;
  }

  .avatar {
    /* Logical sizing */
    inline-size: 2.5rem;
    block-size: 2.5rem;
    border-radius: 50%;
    margin-inline-end: var(--space-sm);
  }

  .breadcrumb-separator {
    /* This arrow flips automatically in RTL */
    margin-inline: var(--space-xs);
  }
}
```

Property mapping reference:

| Physical | Logical (horizontal writing mode) |
|----------|-----------------------------------|
| `width` | `inline-size` |
| `height` | `block-size` |
| `margin-left` | `margin-inline-start` |
| `margin-right` | `margin-inline-end` |
| `padding-top` | `padding-block-start` |
| `padding-bottom` | `padding-block-end` |
| `border-left` | `border-inline-start` |
| `border-right` | `border-inline-end` |
| `top` | `inset-block-start` |
| `left` | `inset-inline-start` |

### Minimal Utility Classes

Define a small set of utility classes (~60) for the overrides that occur frequently. These live in the `utilities` layer so they always win over component styles.

```css
@layer utilities {
  /* Display */
  .hidden { display: none; }
  .block { display: block; }
  .flex { display: flex; }
  .grid { display: grid; }
  .inline-flex { display: inline-flex; }

  /* Flex utilities */
  .items-center { align-items: center; }
  .items-start { align-items: flex-start; }
  .justify-between { justify-content: space-between; }
  .justify-center { justify-content: center; }
  .gap-xs { gap: var(--space-xs); }
  .gap-sm { gap: var(--space-sm); }
  .gap-md { gap: var(--space-md); }
  .gap-lg { gap: var(--space-lg); }

  /* Spacing */
  .mt-sm { margin-block-start: var(--space-sm); }
  .mt-md { margin-block-start: var(--space-md); }
  .mt-lg { margin-block-start: var(--space-lg); }
  .mb-sm { margin-block-end: var(--space-sm); }
  .mb-md { margin-block-end: var(--space-md); }
  .mb-lg { margin-block-end: var(--space-lg); }
  .p-sm { padding: var(--space-sm); }
  .p-md { padding: var(--space-md); }
  .p-lg { padding: var(--space-lg); }

  /* Text */
  .text-sm { font-size: var(--text-sm); }
  .text-lg { font-size: var(--text-lg); }
  .text-xl { font-size: var(--text-xl); }
  .text-center { text-align: center; }
  .text-end { text-align: end; }
  .text-muted { color: var(--color-text-muted); }
  .font-semibold { font-weight: 600; }
  .font-bold { font-weight: 700; }
  .truncate {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  /* Borders and radius */
  .rounded-sm { border-radius: var(--radius-sm); }
  .rounded-md { border-radius: var(--radius-md); }
  .rounded-lg { border-radius: var(--radius-lg); }
  .rounded-full { border-radius: 9999px; }
  .border { border: 1px solid var(--color-border); }

  /* Shadows */
  .shadow-sm { box-shadow: var(--shadow-sm); }
  .shadow-md { box-shadow: var(--shadow-md); }
  .shadow-lg { box-shadow: var(--shadow-lg); }

  /* Widths */
  .w-full { inline-size: 100%; }
  .max-w-sm { max-inline-size: 24rem; }
  .max-w-md { max-inline-size: 28rem; }
  .max-w-lg { max-inline-size: 32rem; }
  .max-w-xl { max-inline-size: 36rem; }

  /* Accessibility */
  .sr-only {
    position: absolute;
    inline-size: 1px;
    block-size: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }

  /* Reduced motion */
  .motion-safe {
    @media (prefers-reduced-motion: reduce) {
      animation: none !important;
      transition: none !important;
    }
  }
}
```

## Full Layer Architecture Example

Here is a complete example showing how all layers work together for a notification badge component:

```css
/* reset layer: box-sizing inherited */
/* base layer: custom properties defined */

@layer components {
  .notification-badge {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-inline-size: 1.25rem;
    block-size: 1.25rem;
    padding-inline: var(--space-xs);
    border-radius: var(--radius-lg);
    font-size: var(--text-sm);
    font-weight: 600;
    background: var(--color-danger);
    color: white;

    /* Empty badge shows as a dot */
    &:empty {
      min-inline-size: 0.5rem;
      block-size: 0.5rem;
      padding: 0;
    }

    /* Enter animation for new notifications */
    @starting-style {
      scale: 0;
      opacity: 0;
    }
    transition: scale 0.2s ease-out, opacity 0.2s ease-out;

    /* Muted variant */
    &.notification-badge--muted {
      background: var(--color-text-muted);
    }
  }
}

@layer modules {
  /* In the header module, badges are positioned absolutely */
  .header-nav .notification-badge {
    position: absolute;
    inset-block-start: -0.25rem;
    inset-inline-end: -0.25rem;
  }
}

@layer utilities {
  /* Utility can override any component or module style */
  .hidden { display: none; }
}
```

```erb
<%# The component styles, module positioning, and utility overrides all layer cleanly %>
<nav class="header-nav">
  <%= link_to notifications_path, class: "nav-link" do %>
    <span class="nav-icon">Bell</span>
    <% if current_user.unread_notifications_count > 0 %>
      <span class="notification-badge"><%= current_user.unread_notifications_count %></span>
    <% end %>
  <% end %>
</nav>
```

## Pattern Card

### GOOD: Layered CSS with OKLCH and Native Nesting

```css
@layer reset, base, components, modules, utilities;

@layer base {
  :root {
    --color-primary: oklch(55% 0.25 260);
    --color-primary-hover: color-mix(in oklch, var(--color-primary), white 15%);
    --color-surface: oklch(99% 0.005 260);
    --color-border: oklch(88% 0.01 260);
  }
}

@layer components {
  .btn {
    padding: var(--space-sm) var(--space-md);
    border-radius: var(--radius-md);
    font-weight: 600;
    transition: background-color 0.15s;

    &.btn--primary {
      background: var(--color-primary);
      color: white;

      &:hover {
        background: var(--color-primary-hover);
      }
    }
  }
}

@layer utilities {
  .w-full { inline-size: 100%; }
}
```

The layer order is declared once. Component styles use nesting for readability. OKLCH colors provide perceptually consistent variants via `color-mix()`. Utilities sit in the highest layer and always override components without `!important`. Adding a new component never risks breaking existing styles because cascade order is explicit.

### BAD: Unsorted CSS or Tailwind Utility Sprawl

```html
<!-- Utility-first: styling is in the HTML, not the stylesheet -->
<button class="px-4 py-2 bg-blue-500 text-white rounded-md font-semibold
  hover:bg-blue-600 active:bg-blue-700 focus:ring-2 focus:ring-blue-500
  focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed
  transition-colors duration-150">
  Save
</button>
```

```css
/* Or: unsorted CSS with specificity fights */
.btn { background: blue; }
.sidebar .btn { background: green; }  /* Override with location */
.btn.btn-primary { background: red; }  /* Override with compound selector */
.page .sidebar .btn.active { background: orange !important; }  /* Nuclear option */
```

Utility-first approaches push all styling decisions into HTML, making templates verbose and design changes a multi-file find-and-replace. Unsorted CSS without layers leads to specificity escalation where every override requires a more specific selector, eventually reaching `!important`. Both problems disappear with explicit `@layer` declarations.
