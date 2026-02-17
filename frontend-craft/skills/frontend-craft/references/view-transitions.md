---
title: "View Transitions API"
categories:
  - ux
  - turbo
  - animation
tags:
  - view-transitions
  - page-transitions
  - turbo-drive
  - css-animation
  - progressive-enhancement
description: >-
  Smooth page transitions using the browser View Transitions API with Turbo.
  Covers meta tag configuration, CSS view-transition-name for element persistence
  across navigations, @view-transition rules, custom animations, and fallback.
---

# View Transitions API

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Enabling View Transitions in Turbo](#enabling-view-transitions-in-turbo)
  - [Element Persistence with view-transition-name](#element-persistence-with-view-transition-name)
  - [Cross-Document View Transitions](#cross-document-view-transitions)
  - [Custom Transition Animations](#custom-transition-animations)
  - [Page-Type Transitions](#page-type-transitions)
  - [Fallback for Unsupported Browsers](#fallback-for-unsupported-browsers)
  - [Accessibility and Reduced Motion](#accessibility-and-reduced-motion)
- [Pattern Card](#pattern-card)

## Overview

The View Transitions API enables smooth animated transitions between page states. When combined with Turbo Drive, it creates fluid navigation experiences where elements visually morph between their old and new positions rather than abruptly swapping.

The API works by capturing a screenshot of the current state, applying the DOM update, then animating between the old screenshot and the new live DOM. This means the browser handles all the complexity of cross-fading, position interpolation, and size changes.

Turbo 8+ has built-in View Transitions support. When enabled, every Turbo Drive navigation automatically uses the View Transitions API if the browser supports it. Elements with matching `view-transition-name` values on both the old and new pages are treated as the "same" element and smoothly morph between positions.

For browsers that do not support View Transitions, the navigation works exactly as before -- a simple body swap with no animation. This makes it a pure progressive enhancement with zero downside.

## Implementation

### Enabling View Transitions in Turbo

Add a meta tag to your layout to enable View Transitions for all Turbo Drive navigations:

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <meta name="view-transition" content="same-origin">
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application" %>
  <%= javascript_importmap_tags %>
</head>
```

That is all that is needed. Turbo will automatically use `document.startViewTransition()` for every navigation when this meta tag is present. The default transition is a cross-fade.

### Element Persistence with view-transition-name

When an element has the same `view-transition-name` on both the source and destination pages, the browser treats it as the same element and animates its position, size, and appearance between the two states.

```css
@layer components {
  /* The page header persists across navigations */
  .site-header {
    view-transition-name: site-header;
  }

  /* Each project card gets a unique transition name */
  .project-card {
    view-transition-name: var(--transition-name);
  }

  /* The main content area has its own transition */
  .main-content {
    view-transition-name: main-content;
  }
}
```

Use inline styles or ERB to set unique transition names for list items:

```erb
<%# app/views/projects/index.html.erb %>
<div class="projects-grid">
  <% @projects.each do |project| %>
    <div class="project-card"
         style="view-transition-name: project-<%= project.id %>">
      <h3><%= link_to project.name, project_path(project) %></h3>
      <p><%= project.description %></p>
    </div>
  <% end %>
</div>
```

```erb
<%# app/views/projects/show.html.erb %>
<article class="project-detail"
         style="view-transition-name: project-<%= @project.id %>">
  <h1><%= @project.name %></h1>
  <div class="project-body"><%= @project.description %></div>
</article>
```

When the user clicks a project card on the index page, the card visually morphs into the detail view. The browser handles all the animation -- position, size, opacity.

**Important rule:** `view-transition-name` values must be unique on a given page. Two elements on the same page cannot share a transition name. The browser will skip the transition if duplicates are found.

### Cross-Document View Transitions

For navigations between different pages (cross-document), use the `@view-transition` at-rule. This is the CSS-only approach that works without JavaScript configuration.

```css
@layer base {
  /* Enable cross-document view transitions */
  @view-transition {
    navigation: auto;
  }
}
```

This is equivalent to the meta tag approach but declared in CSS. Both methods work with Turbo. The meta tag approach is preferred for Turbo applications because Turbo manages the transition lifecycle.

### Custom Transition Animations

Override the default cross-fade with custom animations using the `::view-transition-*` pseudo-elements.

```css
@layer components {
  /* Custom slide transition for main content */
  ::view-transition-old(main-content) {
    animation: slide-out-left 0.25s ease-in;
  }

  ::view-transition-new(main-content) {
    animation: slide-in-right 0.25s ease-out;
  }

  @keyframes slide-out-left {
    from { transform: translateX(0); opacity: 1; }
    to { transform: translateX(-30px); opacity: 0; }
  }

  @keyframes slide-in-right {
    from { transform: translateX(30px); opacity: 0; }
    to { transform: translateX(0); opacity: 1; }
  }

  /* Scale-up transition for project cards */
  ::view-transition-group(project-card) {
    animation-duration: 0.3s;
    animation-timing-function: ease-in-out;
  }

  /* Fade-through for the header (quick fade out, then in) */
  ::view-transition-old(site-header) {
    animation: fade-out 0.1s ease-out;
  }

  ::view-transition-new(site-header) {
    animation: fade-in 0.1s ease-in 0.1s;
  }

  @keyframes fade-out {
    from { opacity: 1; }
    to { opacity: 0; }
  }

  @keyframes fade-in {
    from { opacity: 0; }
    to { opacity: 1; }
  }
}
```

The View Transitions API creates three pseudo-elements for each named transition:

| Pseudo-Element | Purpose |
|----------------|---------|
| `::view-transition-group(name)` | Wrapper that animates size and position between old and new |
| `::view-transition-old(name)` | Screenshot of the old state, animates out |
| `::view-transition-new(name)` | Live new DOM element, animates in |

### Page-Type Transitions

Use different transitions for different types of navigation. For example, slide forward when drilling into a detail page, and slide back when returning to a list.

```css
@layer components {
  /* Default transition: cross-fade */
  ::view-transition-old(main-content),
  ::view-transition-new(main-content) {
    animation-duration: 0.2s;
  }

  /* Forward navigation: slide left */
  html[data-transition="forward"] {
    & ::view-transition-old(main-content) {
      animation: slide-out-left 0.25s ease-in forwards;
    }

    & ::view-transition-new(main-content) {
      animation: slide-in-right 0.25s ease-out forwards;
    }
  }

  /* Back navigation: slide right */
  html[data-transition="back"] {
    & ::view-transition-old(main-content) {
      animation: slide-out-right 0.25s ease-in forwards;
    }

    & ::view-transition-new(main-content) {
      animation: slide-in-left 0.25s ease-out forwards;
    }
  }

  @keyframes slide-out-right {
    to { transform: translateX(30px); opacity: 0; }
  }

  @keyframes slide-in-left {
    from { transform: translateX(-30px); opacity: 0; }
  }
}
```

Set the transition direction with a Stimulus controller:

```javascript
// app/javascript/controllers/navigation_direction_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    document.addEventListener("turbo:before-visit", this.setDirection)
  }

  disconnect() {
    document.removeEventListener("turbo:before-visit", this.setDirection)
  }

  setDirection = (event) => {
    const currentPath = window.location.pathname
    const targetPath = new URL(event.detail.url).pathname

    // Deeper path = forward, shallower = back
    const direction = targetPath.split("/").length > currentPath.split("/").length
      ? "forward"
      : "back"

    document.documentElement.dataset.transition = direction
  }
}
```

### Fallback for Unsupported Browsers

View Transitions are a progressive enhancement. Always ensure the app works without them. Use `@supports` to scope View Transition styles.

```css
@layer components {
  /* Only apply view-transition-name when the API is supported */
  @supports (view-transition-name: none) {
    .project-card {
      view-transition-name: var(--transition-name);
    }

    .site-header {
      view-transition-name: site-header;
    }

    .main-content {
      view-transition-name: main-content;
    }

    ::view-transition-old(main-content) {
      animation: fade-out 0.15s ease-out;
    }

    ::view-transition-new(main-content) {
      animation: fade-in 0.15s ease-in;
    }
  }
}
```

For JavaScript-driven transitions, use optional chaining:

```javascript
// Safe to call in any browser
document.startViewTransition?.(() => {
  // DOM update here
  targetElement.innerHTML = newContent
})

// Or with a callback fallback
if (document.startViewTransition) {
  document.startViewTransition(() => updateDOM())
} else {
  updateDOM()
}
```

### Accessibility and Reduced Motion

Always disable or simplify transitions for users who prefer reduced motion.

```css
@layer components {
  @media (prefers-reduced-motion: reduce) {
    ::view-transition-group(*),
    ::view-transition-old(*),
    ::view-transition-new(*) {
      animation-duration: 0.01ms !important;
    }
  }
}
```

This collapses all view transition animations to near-instant, preserving the DOM update but eliminating the visual motion.

## Pattern Card

### GOOD: CSS View Transitions with Turbo and Fallback

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <meta name="view-transition" content="same-origin">
</head>
<body>
  <header class="site-header">
    <%= render "shared/navigation" %>
  </header>

  <main class="main-content">
    <%= yield %>
  </main>
</body>
```

```css
@layer components {
  @supports (view-transition-name: none) {
    .site-header { view-transition-name: site-header; }
    .main-content { view-transition-name: main-content; }

    ::view-transition-old(main-content) {
      animation: fade-out 0.15s ease-out;
    }

    ::view-transition-new(main-content) {
      animation: fade-in 0.15s ease-in;
    }

    @keyframes fade-out {
      from { opacity: 1; }
      to { opacity: 0; }
    }

    @keyframes fade-in {
      from { opacity: 0; }
      to { opacity: 1; }
    }
  }

  @media (prefers-reduced-motion: reduce) {
    ::view-transition-group(*),
    ::view-transition-old(*),
    ::view-transition-new(*) {
      animation-duration: 0.01ms !important;
    }
  }
}
```

One meta tag enables View Transitions for all Turbo navigations. Named elements persist across pages with smooth morphing. The `@supports` block ensures no broken styles in unsupported browsers. Reduced motion is respected. No JavaScript animation library is needed.

### BAD: JavaScript-Driven Page Transitions

```javascript
// A JavaScript library to animate page transitions
import Barba from "@barba/core"
import gsap from "gsap"

Barba.init({
  transitions: [{
    async leave(data) {
      await gsap.to(data.current.container, {
        opacity: 0,
        y: -20,
        duration: 0.3
      })
    },
    async enter(data) {
      await gsap.from(data.next.container, {
        opacity: 0,
        y: 20,
        duration: 0.3
      })
    }
  }]
})
```

This approach adds two JavaScript dependencies (Barba.js + GSAP), fights with Turbo's navigation lifecycle, requires careful teardown to avoid memory leaks, does not benefit from browser-level optimization, and breaks if JavaScript fails to load. The View Transitions API achieves the same result with zero JavaScript and a few lines of CSS, while Turbo handles all the page lifecycle coordination automatically.
