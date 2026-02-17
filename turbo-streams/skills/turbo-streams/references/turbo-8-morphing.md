---
title: Turbo 8 Morphing for Page Updates
date: 2025-02-10
categories:
  - Turbo Streams
tags:
  - turbo-8
  - morphing
  - idiomorph
  - data-turbo-permanent
  - page-refresh
  - dom-preservation
description: Use DOM morphing to preserve form state, scroll position, and focus during page updates.
---

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Enabling Page Morphing](#enabling-page-morphing)
  - [How Morphing Works](#how-morphing-works)
  - [Stream Action Morph](#stream-action-morph)
  - [Permanent Elements](#permanent-elements)
  - [Frame Morphing](#frame-morphing)
  - [Morph Lifecycle Events](#morph-lifecycle-events)
  - [Stimulus Controller Considerations](#stimulus-controller-considerations)
- [Key Points](#key-points)
- [Pattern Card: Morph Preserving State](#pattern-card-morph-preserving-state)

## Overview

Turbo 8 introduced DOM morphing powered by Idiomorph. Instead of swapping the entire `<body>` or replacing elements wholesale, morphing compares the old and new DOM trees and applies the minimal set of changes needed. This preserves focus, scroll position, form input values, CSS transitions, and other ephemeral DOM state that would be lost with a full replacement.

Morphing is available in two forms:
1. **Page refresh morphing** -- morph the entire page on navigation using `turbo_refreshes_with method: :morph`
2. **Stream action morphing** -- morph a specific element via `turbo_stream.action(:morph, target, ...)`

## Implementation

### Enabling Page Morphing

Add `turbo_refreshes_with` to your layout to opt into morphing for all page refreshes:

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
    <%# ... %>
  </head>
  <body>
    <%= yield %>
  </body>
</html>
```

This inserts two `<meta>` tags:

```html
<meta name="turbo-refresh-method" content="morph">
<meta name="turbo-refresh-scroll" content="preserve">
```

- `method: :morph` -- use Idiomorph to diff and patch the DOM instead of replacing the body
- `scroll: :preserve` -- maintain scroll position after the morph (default is to reset to top)

### How Morphing Works

Idiomorph walks the old and new DOM trees simultaneously:

1. Elements are matched by `id` attribute first, then by tag name and position
2. Matching elements have their attributes and text content updated in place
3. New elements are inserted, missing elements are removed
4. Elements with `data-turbo-permanent` are skipped entirely
5. The algorithm minimizes DOM mutations for efficiency

Because elements are updated rather than replaced, the following state is preserved:
- Focus on the active input field
- Text typed into form fields (unless the server sends different values)
- Scroll position within scrollable containers
- CSS animation/transition state
- Event listeners attached directly to DOM elements
- Open `<details>` elements

### Stream Action Morph

Use the `morph` action in Turbo Stream responses to morph a specific target element:

```erb
<%# app/views/tasks/update.turbo_stream.erb %>
<%= turbo_stream.action(:morph, dom_id(@task)) do %>
  <%= render partial: "tasks/task", locals: { task: @task } %>
<% end %>
```

Or using the shorthand if your Turbo version supports it:

```erb
<%= turbo_stream.morph dom_id(@task), partial: "tasks/task", locals: { task: @task } %>
```

Stream morph is useful when you want to update a complex element (like a card with nested interactive components) without losing the state of its children.

### Permanent Elements

Mark elements with `data-turbo-permanent` to exclude them from morphing entirely. The element and all its children will be left untouched:

```erb
<%# A chat widget that maintains its own state %>
<div id="chat-widget" data-turbo-permanent>
  <%= render "shared/chat_widget" %>
</div>

<%# A video player that should keep playing %>
<div id="video-player" data-turbo-permanent>
  <video src="<%= @video.url %>" autoplay></video>
</div>

<%# A dropdown menu that should stay open %>
<div id="user-menu" data-turbo-permanent>
  <%= render "shared/user_dropdown" %>
</div>
```

Requirements for `data-turbo-permanent`:
- The element **must** have a unique `id` attribute
- The element must exist in both the old and new page
- If the element is missing from the new page, it will be removed

### Frame Morphing

Turbo Frames can opt into morphing when they refresh:

```erb
<%= turbo_frame_tag "task_list", refresh: :morph do %>
  <%= render @tasks %>
<% end %>
```

When combined with page-level morphing, frame morphing ensures that navigating within a frame also preserves DOM state. This is especially useful for paginated lists or tab-switching interfaces where you want to preserve user interactions within the frame.

### Morph Lifecycle Events

Turbo fires events during the morph process that you can hook into:

```javascript
// Fired before each element is morphed
document.addEventListener("turbo:before-morph-element", (event) => {
  const { target, newElement } = event.detail

  // Prevent morphing a specific element
  if (target.id === "keep-this-unchanged") {
    event.preventDefault()
  }
})

// Fired after each element is morphed
document.addEventListener("turbo:morph-element", (event) => {
  const { target, newElement } = event.detail

  // Re-initialize third-party libraries after morph
  if (target.matches("[data-chart]")) {
    initializeChart(target)
  }
})

// Fired before the entire morph begins
document.addEventListener("turbo:before-morph", (event) => {
  // Save ephemeral state before morph
  window._savedScrollPositions = {}
  document.querySelectorAll("[data-save-scroll]").forEach(el => {
    window._savedScrollPositions[el.id] = el.scrollTop
  })
})

// Fired after the entire morph completes
document.addEventListener("turbo:morph", (event) => {
  // Restore ephemeral state after morph
  if (window._savedScrollPositions) {
    Object.entries(window._savedScrollPositions).forEach(([id, scrollTop]) => {
      const el = document.getElementById(id)
      if (el) el.scrollTop = scrollTop
    })
  }
})
```

### Stimulus Controller Considerations

Stimulus controllers are generally preserved during morphing because their host elements are updated in place rather than replaced. However, there are edge cases:

```javascript
// app/javascript/controllers/timer_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.interval = setInterval(() => this.tick(), 1000)
  }

  disconnect() {
    clearInterval(this.interval)
  }

  // Handle morph explicitly if needed
  morphElement({ detail: { newElement } }) {
    // Called when this controller's element is morphed
    // Useful for reconciling state
  }
}
```

If a morph causes a Stimulus controller to disconnect and reconnect, the `disconnect()` and `connect()` callbacks fire normally. Use these to clean up and re-initialize state.

For controllers that manage timers, WebSocket connections, or other stateful resources, listen to `turbo:morph-element` to handle the transition:

```javascript
document.addEventListener("turbo:morph-element", (event) => {
  const controller = event.detail.target
    .closest("[data-controller~='countdown']")
    ?.__stimulusController

  if (controller) {
    controller.restartTimer()
  }
})
```

## Key Points

- Enable page morphing with `turbo_refreshes_with method: :morph, scroll: :preserve`
- Morphing preserves focus, scroll position, form inputs, and CSS state
- Use `data-turbo-permanent` on elements that must never be touched
- Permanent elements require a unique `id` and must exist in both old and new DOM
- Use `turbo:before-morph-element` to prevent morphing of specific elements
- Use `turbo:morph-element` to re-initialize third-party libraries after morph
- Stimulus controllers persist through morphing but may need explicit state management
- Frame morphing with `refresh: :morph` preserves state within Turbo Frames

## Pattern Card: Morph Preserving State

**When to use**: Update a complex page section while preserving user-entered form data, scroll position, focus, and interactive widget state.

**GOOD - Morph preserves DOM state during updates**:

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
</head>
```

```erb
<%# app/views/tasks/update.turbo_stream.erb %>
<%= turbo_stream.action(:morph, dom_id(@task)) do %>
  <%= render partial: "tasks/task", locals: { task: @task } %>
<% end %>
```

```erb
<%# Permanent elements survive morphing %>
<div id="chat-widget" data-turbo-permanent>
  <%= render "shared/chat" %>
</div>
```

User is editing a form field, a background broadcast morphs the page, and the form field retains its value and focus.

**BAD - Replace loses user input and scroll position**:

```erb
<%# app/views/tasks/update.turbo_stream.erb %>
<%= turbo_stream.replace "task_section" do %>
  <%= render "tasks/full_section", tasks: @tasks %>
<% end %>
```

User is typing in a search field inside `task_section`. The replace wipes out their input and resets scroll to the top.
