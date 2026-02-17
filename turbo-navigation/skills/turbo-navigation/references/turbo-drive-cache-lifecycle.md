---
title: "Turbo Drive Cache Lifecycle"
categories:
  - turbo
  - navigation
  - caching
tags:
  - turbo-drive
  - cache
  - snapshot
  - preview
  - page-lifecycle
description: >-
  How Turbo Drive intercepts link clicks, caches page snapshots, and restores
  them for instant preview navigations. Covers Cache-Control headers, snapshot
  cleanup, the turbo:before-cache event, and stale data prevention strategies.
---

# Turbo Drive Cache Lifecycle

## Table of Contents

- [Overview](#overview)
- [How Turbo Drive Navigation Works](#how-turbo-drive-navigation-works)
- [Cache Snapshot Lifecycle](#cache-snapshot-lifecycle)
- [Implementation](#implementation)
  - [Cache-Control Headers](#cache-control-headers)
  - [Disabling Cache Per Page](#disabling-cache-per-page)
  - [Cleaning Up Before Caching](#cleaning-up-before-caching)
  - [Clearing the Cache Programmatically](#clearing-the-cache-programmatically)
- [Handling Stale Data](#handling-stale-data)
- [Pattern Card](#pattern-card)

## Overview

Turbo Drive intercepts every link click and form submission within your application, fetching the new page via `fetch()` and swapping the `<body>` content without a full browser reload. To make back/forward navigation feel instant, Turbo Drive caches a snapshot of each page before navigating away. When the user navigates back, the cached snapshot is shown immediately as a "preview" while a fresh copy is fetched in the background.

This caching behavior is powerful but requires careful management. Stale data, leftover flash messages, open modals, and ephemeral UI state can all leak into cache snapshots and confuse users when they navigate back.

## How Turbo Drive Navigation Works

1. User clicks a link or submits a form.
2. Turbo Drive fires `turbo:before-visit` -- you can cancel navigation here.
3. Turbo Drive takes a snapshot of the current page and stores it in the cache.
4. The Turbo progress bar appears (if the response takes > 500ms).
5. Turbo Drive fetches the new page via `fetch()`.
6. The response `<body>` replaces the current `<body>`.
7. Turbo Drive fires `turbo:load` -- the new page is ready.

On back/forward navigation:
1. Turbo Drive restores the cached snapshot immediately (preview visit).
2. Turbo Drive fetches a fresh copy from the server in the background.
3. When the fresh copy arrives, it replaces the preview.

## Cache Snapshot Lifecycle

```
User clicks link
       |
       v
turbo:before-visit  (cancelable)
       |
       v
turbo:before-cache  (clean up ephemeral UI here)
       |
       v
Snapshot stored in cache
       |
       v
turbo:visit  (navigation begins)
       |
       v
turbo:before-render  (new page about to render)
       |
       v
turbo:render  (new page rendered)
       |
       v
turbo:load  (navigation complete)
```

## Implementation

### Cache-Control Headers

Turbo Drive respects standard HTTP `Cache-Control` headers. If a response includes `no-cache` or `no-store`, Turbo Drive will not cache the page snapshot.

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  private

  # Call this in controllers where pages should never be cached by Turbo Drive.
  def disable_turbo_cache
    response.headers["Cache-Control"] = "no-cache, no-store"
  end
end
```

```ruby
# app/controllers/checkout_controller.rb
class CheckoutController < ApplicationController
  before_action :disable_turbo_cache

  def show
    @order = Current.user.pending_order
  end
end
```

### Disabling Cache Per Page

Use a `<meta>` tag to tell Turbo Drive not to cache a specific page. This is useful when the page contains sensitive or highly dynamic content.

```erb
<%# app/views/admin/dashboard/show.html.erb %>
<% content_for :head do %>
  <meta name="turbo-cache-control" content="no-cache">
<% end %>

<h1>Admin Dashboard</h1>
<%# ... real-time stats that should always be fresh ... %>
```

Or use the `data-turbo-cache` attribute on individual elements to exclude them from the snapshot:

```erb
<div data-turbo-cache="false">
  <%# This entire subtree will be removed from the cached snapshot %>
  <div class="flash-messages">
    <% flash.each do |type, message| %>
      <div class="flash flash-<%= type %>"><%= message %></div>
    <% end %>
  </div>
</div>
```

### Cleaning Up Before Caching

The `turbo:before-cache` event fires just before Turbo Drive takes the page snapshot. Use it to remove ephemeral UI elements that should not appear when the user navigates back.

```javascript
// app/javascript/application.js
document.addEventListener("turbo:before-cache", () => {
  // Remove flash messages so they don't reappear on back navigation
  document.querySelectorAll(".flash").forEach(el => el.remove())

  // Close any open modals
  document.querySelectorAll("[data-modal].open").forEach(modal => {
    modal.classList.remove("open")
  })

  // Reset form states
  document.querySelectorAll("form").forEach(form => form.reset())
})
```

### Clearing the Cache Programmatically

When data changes significantly (e.g., after a bulk action), you may want to clear the Turbo Drive cache entirely so stale previews are never shown.

```javascript
// Clear the entire Turbo Drive page cache
Turbo.cache.clear()
```

You can trigger this from a Stimulus controller after an action completes:

```javascript
// app/javascript/controllers/bulk_action_controller.js
import { Controller } from "@hotwired/stimulus"
import { Turbo } from "@hotwired/turbo-rails"

export default class extends Controller {
  afterComplete() {
    Turbo.cache.clear()
    Turbo.visit(window.location.href, { action: "replace" })
  }
}
```

## Handling Stale Data

The most common source of confusion with Turbo Drive caching is stale data appearing briefly during preview visits. Here are strategies to mitigate this:

1. **Mark volatile sections with `data-turbo-cache="false"`.** Content that changes frequently (notification counts, real-time data) should be excluded from the cache.

2. **Use `turbo:before-cache` to clean up.** Remove flash messages, close modals, reset forms.

3. **Add a visual indicator during preview visits.** Turbo adds `data-turbo-preview` to the `<html>` element during preview visits. Use CSS to show a subtle loading indicator:

```css
/* app/assets/stylesheets/turbo.css */
html[data-turbo-preview] {
  opacity: 0.9;
  pointer-events: none;
}

html[data-turbo-preview] .turbo-preview-indicator {
  display: block;
}
```

4. **Disable caching for pages with forms or sensitive data.** Checkout flows, admin dashboards, and multi-step wizards should use `no-cache`.

## Pattern Card

### GOOD: Preview with Loading Indicator and Cache Cleanup

```erb
<%# app/views/layouts/application.html.erb %>
<html>
<head>
  <meta name="turbo-cache-control" content="no-preview">
  <%# Or allow preview but clean up: %>
</head>
<body>
  <div data-turbo-cache="false">
    <% flash.each do |type, message| %>
      <div class="flash flash-<%= type %>"><%= message %></div>
    <% end %>
  </div>

  <div class="turbo-preview-indicator" style="display: none;">
    Loading fresh data...
  </div>

  <%= yield %>
</body>
</html>
```

```javascript
// Clean up before caching
document.addEventListener("turbo:before-cache", () => {
  document.querySelectorAll(".tooltip, .dropdown.open").forEach(el => el.remove())
})
```

This approach ensures cached previews never show stale flash messages or ephemeral UI, and the user sees a subtle indicator that fresh data is loading.

### BAD: No Cache Cleanup Leads to Stale Data

```erb
<%# app/views/layouts/application.html.erb %>
<body>
  <%# Flash messages will reappear on back navigation %>
  <% flash.each do |type, message| %>
    <div class="flash flash-<%= type %>"><%= message %></div>
  <% end %>

  <%# Open modals will be cached in their open state %>
  <div id="confirm-modal" class="modal open">
    <p>Are you sure?</p>
  </div>

  <%= yield %>
</body>
```

Without cache cleanup, users navigating back see stale flash messages ("Item saved!") that are no longer relevant, and modals frozen in their open state. This creates a confusing and broken-feeling experience.
