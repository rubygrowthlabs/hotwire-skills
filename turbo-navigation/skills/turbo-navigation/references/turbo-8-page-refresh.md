---
title: "Turbo 8 Page Refresh"
categories:
  - turbo
  - navigation
  - turbo-8
tags:
  - turbo-8
  - morph
  - page-refresh
  - scroll-preservation
  - permanent-elements
description: >-
  Using Turbo 8 morphing page refresh to update pages while preserving scroll
  position and DOM state. Covers the turbo_refreshes_with method, morph vs replace
  strategies, permanent elements, and integration with Turbo Streams broadcasts.
---

# Turbo 8 Page Refresh

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Enabling Morph Refresh in the Layout](#enabling-morph-refresh-in-the-layout)
  - [Per-Page Refresh Configuration](#per-page-refresh-configuration)
  - [Morph vs Replace: When to Use Each](#morph-vs-replace-when-to-use-each)
  - [Permanent Elements](#permanent-elements)
  - [Integration with Turbo Streams Broadcasts](#integration-with-turbo-streams-broadcasts)
  - [Handling Stimulus Controllers During Morph](#handling-stimulus-controllers-during-morph)
- [Pattern Card](#pattern-card)

## Overview

Turbo 8 introduced page refresh with morphing, a new way to update pages that uses DOM morphing (via idiomorph) instead of full body replacement. When a page refresh is triggered, Turbo fetches a fresh copy of the current page and morphs the existing DOM to match, preserving scroll position, focus state, form input values, and CSS transitions.

This is controlled by two settings:

- **`method`**: `:morph` (diff and patch the DOM) or `:replace` (replace the entire `<body>`, the Turbo Drive default).
- **`scroll`**: `:preserve` (keep current scroll position) or `:reset` (scroll to top).

The primary use case is combining page refresh with Turbo Streams broadcasts. When the server broadcasts a `turbo_stream_action_tag "refresh"`, Turbo fetches a fresh copy of the current page and morphs it in. This is simpler than crafting individual Turbo Stream actions for each DOM change -- you just tell the page to refresh and the server re-renders everything from scratch.

## Implementation

### Enabling Morph Refresh in the Layout

Set the refresh method and scroll behavior in your application layout. This applies to all pages unless overridden.

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
  <%= yield :head %>
  <%# ... other head content ... %>
</head>
<body>
  <%= yield %>
</body>
</html>
```

The `turbo_refreshes_with` helper generates `<meta>` tags that Turbo reads:

```html
<meta name="turbo-refresh-method" content="morph">
<meta name="turbo-refresh-scroll" content="preserve">
```

### Per-Page Refresh Configuration

Override the layout default for specific pages using `content_for`:

```erb
<%# app/views/checkout/show.html.erb %>
<%# Checkout should always replace fully and scroll to top %>
<% content_for :head do %>
  <%= turbo_refreshes_with method: :replace, scroll: :reset %>
<% end %>

<h1>Checkout</h1>
<%# ... checkout form ... %>
```

```erb
<%# app/views/conversations/show.html.erb %>
<%# Chat page should morph to preserve scroll in the message list %>
<% content_for :head do %>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
<% end %>

<div class="conversation">
  <div class="messages" id="messages">
    <%= render @messages %>
  </div>
  <%= render "message_form", conversation: @conversation %>
</div>
```

### Morph vs Replace: When to Use Each

| Scenario | Method | Scroll | Reason |
|----------|--------|--------|--------|
| Chat / messaging page | `:morph` | `:preserve` | Preserve scroll position in message list |
| Dashboard with live data | `:morph` | `:preserve` | Update stats without losing scroll context |
| Form page after validation error | `:morph` | `:preserve` | Keep user's scroll position near the error |
| Multi-step wizard | `:replace` | `:reset` | Each step is conceptually a new page |
| Checkout flow | `:replace` | `:reset` | Security-sensitive; avoid stale DOM state |
| Search results after filter change | `:replace` | `:reset` | Results changed entirely; scroll to top |
| Content feed with new items | `:morph` | `:preserve` | New items appear without disrupting reading |

### Permanent Elements

Some elements should never be morphed or replaced during a page refresh. Mark them with `data-turbo-permanent` and a unique `id`. Turbo will skip these elements during morphing.

Common use cases:
- Audio/video players that should keep playing.
- Third-party widget embeds (maps, charts).
- Elements with complex client-side state that is hard to restore.

```erb
<%# app/views/layouts/application.html.erb %>
<body>
  <header>
    <%# Navigation bar is permanent -- active states and dropdowns persist %>
    <nav id="main-nav" data-turbo-permanent>
      <%= render "shared/navigation" %>
    </nav>
  </header>

  <main>
    <%= yield %>
  </main>

  <%# Audio player should keep playing during morph refreshes %>
  <div id="audio-player" data-turbo-permanent>
    <%= render "shared/audio_player" %>
  </div>

  <%# Flash messages should NOT be permanent -- they should update %>
  <div id="flash-messages">
    <%= render "shared/flash" %>
  </div>
</body>
```

**Important rules for permanent elements:**

1. The element must have a unique `id` attribute. Turbo matches permanent elements by ID.
2. The element must exist in both the current page and the new response. If Turbo cannot find the matching ID in the new response, the element may be removed.
3. Use permanent elements sparingly. They create islands of stale DOM that do not update, which can lead to inconsistencies.

### Integration with Turbo Streams Broadcasts

The most powerful use of Turbo 8 page refresh is combining it with Turbo Streams broadcasts. Instead of crafting specific stream actions (`append`, `replace`, `remove`) for every model change, you broadcast a simple "refresh" action. The client then fetches and morphs the entire page.

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :conversation

  after_create_commit -> {
    broadcast_refresh_to conversation
  }

  after_update_commit -> {
    broadcast_refresh_to conversation
  }

  after_destroy_commit -> {
    broadcast_refresh_to conversation
  }
end
```

```erb
<%# app/views/conversations/show.html.erb %>
<%= turbo_stream_from @conversation %>

<div class="conversation">
  <div class="messages">
    <%= render @messages %>
  </div>
</div>
```

When a new message is created, `broadcast_refresh_to` sends a Turbo Stream refresh action to all subscribers. Each client fetches a fresh copy of the page and morphs the DOM, adding the new message while preserving scroll position and form state.

This is simpler than the Turbo 7 approach of broadcasting specific `append` or `replace` actions, which required matching the exact DOM structure and could break if the view changed.

**Debouncing refreshes:**

When multiple model changes happen in quick succession (e.g., a bulk operation), you do not want to trigger a refresh for each one. Use `broadcast_refresh_later_to` for debounced refreshes:

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :project

  # Debounced -- multiple rapid changes result in a single refresh
  after_create_commit  -> { broadcast_refresh_later_to project }
  after_update_commit  -> { broadcast_refresh_later_to project }
  after_destroy_commit -> { broadcast_refresh_later_to project }
end
```

### Handling Stimulus Controllers During Morph

When Turbo morphs the DOM, Stimulus controllers on morphed elements go through a reconnection cycle. The controller's `connect()` method fires again if the element is re-added or its attributes change. However, `disconnect()` may also fire, which can cause issues with controllers that manage external resources.

Use `data-turbo-permanent` for elements with controllers that hold important client-side state:

```erb
<%# This chart controller maintains complex internal state %>
<div id="revenue-chart"
     data-turbo-permanent
     data-controller="chart"
     data-chart-url-value="<%= revenue_data_path %>">
</div>
```

For controllers that need to survive morph without `data-turbo-permanent`, check for existing state in `connect()`:

```javascript
// app/javascript/controllers/editor_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    // Only initialize if not already set up (handles morph reconnection)
    if (!this.initialized) {
      this.editor = this.createEditor()
      this.initialized = true
    }
  }

  disconnect() {
    // Only tear down on actual removal, not morph disconnection.
    // Turbo morphing triggers disconnect/connect rapidly.
    // Use a timeout to distinguish real disconnects from morph cycles.
    this.disconnectTimeout = setTimeout(() => {
      this.editor?.destroy()
      this.initialized = false
    }, 100)
  }

  // Cancel teardown if reconnected quickly (morph cycle)
  reconnect() {
    clearTimeout(this.disconnectTimeout)
  }
}
```

## Pattern Card

### GOOD: Morph Refresh Preserving Scroll and DOM State

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
</head>
<body>
  <nav id="main-nav" data-turbo-permanent>
    <%= render "shared/navigation" %>
  </nav>

  <main>
    <%= yield %>
  </main>
</body>
```

```erb
<%# app/views/projects/show.html.erb %>
<%= turbo_stream_from @project %>

<div class="project-detail">
  <h1><%= @project.name %></h1>

  <div class="tasks-list">
    <%= render @project.tasks %>
  </div>
</div>
```

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :project
  after_create_commit  -> { broadcast_refresh_later_to project }
  after_update_commit  -> { broadcast_refresh_later_to project }
  after_destroy_commit -> { broadcast_refresh_later_to project }
end
```

When a task is created, updated, or deleted, the page morphs to reflect the change. The user's scroll position, open dropdowns, and form inputs are preserved. The navigation bar is marked permanent so its state (active classes, dropdowns) never resets. Debounced broadcasts prevent multiple rapid changes from triggering excessive refreshes.

### BAD: Full Page Replace Losing DOM State

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <%# No turbo_refreshes_with -- defaults to replace %>
</head>
<body>
  <nav>
    <%= render "shared/navigation" %>
  </nav>
  <main>
    <%= yield %>
  </main>
</body>
```

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :project
  # Broadcasting specific stream actions for every change
  after_create_commit  -> { broadcast_append_to project, target: "tasks" }
  after_update_commit  -> { broadcast_replace_to project }
  after_destroy_commit -> { broadcast_remove_to project }
end
```

Without morph refresh, each broadcast must specify the exact stream action and target. This is fragile -- if the view structure changes, the broadcast breaks. Using `replace` as the refresh method (the default) causes the entire body to swap, losing scroll position, resetting form fields, and potentially disrupting CSS transitions. Individual stream actions (`append`, `replace`, `remove`) are harder to maintain and require keeping the model's broadcast logic in sync with the view's DOM structure.
