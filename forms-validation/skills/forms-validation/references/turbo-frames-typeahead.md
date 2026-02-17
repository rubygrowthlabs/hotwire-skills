---
title: "Typeahead Search with Turbo Frames"
categories:
  - turbo
  - forms
  - search
tags:
  - turbo-frames
  - typeahead
  - debounce
  - auto-submit
  - search
  - loading-state
description: >-
  As-you-type search updating results in a Turbo Frame. Covers debounced
  auto-submit with Stimulus, loading states via aria-busy, URL parameter
  preservation for bookmarkable search, and graceful degradation.
---

# Typeahead Search with Turbo Frames

## Table of Contents

- [Overview](#overview)
- [Architecture: Form + Frame + Auto-Submit](#architecture-form--frame--auto-submit)
- [Implementation](#implementation)
  - [The Search Form](#the-search-form)
  - [The Results Frame](#the-results-frame)
  - [The Auto-Submit Stimulus Controller](#the-auto-submit-stimulus-controller)
  - [The Controller Action](#the-controller-action)
  - [Debouncing Input](#debouncing-input)
  - [Loading States](#loading-states)
  - [URL Parameter Preservation](#url-parameter-preservation)
- [Combining Search with Filters](#combining-search-with-filters)
- [Pattern Card](#pattern-card)

## Overview

Typeahead search lets users see results update as they type, without submitting a form or navigating to a new page. With Turbo Frames, the search form submits via GET to an action that renders results inside a targeted frame. A Stimulus controller auto-submits the form on input with debouncing to avoid overwhelming the server.

This pattern is ideal for:
- Filtering a list of records as the user types
- Command palette / quick search interfaces
- Autocomplete dropdowns
- Any search interface that should feel instant

## Architecture: Form + Frame + Auto-Submit

```
Search form (GET)              Server
+---------------------------+  +---------------------------+
| [Search: "buy gro____"]  |  | TasksController#index     |
|   data-turbo-frame=       |  |   @tasks = Task.search(q) |
|     "tasks_results"       |  |   render partial results  |
|                           |  +---------------------------+
| turbo_frame "tasks_results"|         |
|   Task 1: Buy groceries   |  <------+
|   Task 3: Buy grout       |  HTML response with
+---------------------------+  matching turbo_frame
```

The form uses `method: :get` and `data-turbo-frame` to target the results frame. Each keystroke (debounced) submits the form, and Turbo replaces only the results frame content.

## Implementation

### The Search Form

The search form submits via GET and targets the results frame. It lives outside the frame so it is not replaced when results update.

```erb
<%# app/views/tasks/index.html.erb %>
<%= form_with url: tasks_path,
      method: :get,
      data: {
        turbo_frame: "tasks_results",
        controller: "auto-submit",
        auto_submit_delay_value: 300
      } do |f| %>
  <div class="relative">
    <%= f.search_field :q,
          value: params[:q],
          placeholder: "Search tasks...",
          autofocus: true,
          autocomplete: "off",
          data: { action: "input->auto-submit#submit" },
          class: "w-full rounded-md border-gray-300 pl-10 shadow-sm focus:border-blue-500 focus:ring-blue-500" %>
    <div class="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
      <%# Search icon SVG %>
    </div>
  </div>
<% end %>
```

Key details:
- `method: :get` -- search forms should use GET so results are bookmarkable.
- `data-turbo-frame="tasks_results"` -- scopes the response to the results frame.
- `data-controller="auto-submit"` -- Stimulus controller handles auto-submission.
- `value: params[:q]` -- preserves the search term across page loads.

### The Results Frame

The results frame wraps only the search results. When the form submits, Turbo replaces this frame's content with the matching frame from the response.

```erb
<%# app/views/tasks/index.html.erb (continued) %>
<%= turbo_frame_tag "tasks_results" do %>
  <% if @tasks.any? %>
    <ul class="divide-y divide-gray-200">
      <% @tasks.each do |task| %>
        <li class="py-3">
          <%= render task %>
        </li>
      <% end %>
    </ul>
  <% else %>
    <p class="py-8 text-center text-gray-500">No tasks found.</p>
  <% end %>
<% end %>
```

### The Auto-Submit Stimulus Controller

This controller submits the form on input events with configurable debouncing.

```javascript
// app/javascript/controllers/auto_submit_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    delay: { type: Number, default: 300 }
  }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }

  // Clean up timeout if controller disconnects
  disconnect() {
    clearTimeout(this.timeout)
  }
}
```

Note: `requestSubmit()` is essential here. It fires the `submit` event that Turbo listens for, unlike `submit()` which bypasses Turbo.

### The Controller Action

The controller filters results based on the search parameter. The same action handles both the initial page load and search requests.

```ruby
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  def index
    @tasks = Task.all
    @tasks = @tasks.search(params[:q]) if params[:q].present?
    @tasks = @tasks.order(created_at: :desc)
  end
end
```

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  scope :search, ->(query) {
    where("title ILIKE :q OR description ILIKE :q", q: "%#{sanitize_sql_like(query)}%")
  }
end
```

### Debouncing Input

The auto-submit controller includes built-in debouncing via the `delay` value. Adjust the delay based on your needs:

- **150-200ms**: Fast local filtering or small datasets
- **300ms**: Good default for server-side search
- **500ms**: Heavy queries or rate-limited APIs

```erb
<%# Fast debounce for small lists %>
<%= form_with url: tasks_path,
      data: { controller: "auto-submit", auto_submit_delay_value: 150 } do |f| %>
```

### Loading States

Turbo Frames automatically add `aria-busy="true"` and a `busy` class while loading. Use CSS to show a loading indicator:

```css
/* app/assets/stylesheets/search.css */
turbo-frame[aria-busy="true"] {
  opacity: 0.6;
  pointer-events: none;
  position: relative;
}

turbo-frame[aria-busy="true"]::after {
  content: "";
  position: absolute;
  top: 50%;
  left: 50%;
  width: 1.5rem;
  height: 1.5rem;
  margin: -0.75rem 0 0 -0.75rem;
  border: 2px solid #3b82f6;
  border-top-color: transparent;
  border-radius: 50%;
  animation: spin 0.6s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}
```

No JavaScript needed for the loading state -- Turbo handles the `aria-busy` attribute lifecycle automatically.

### URL Parameter Preservation

To make search results bookmarkable and shareable, update the browser URL as the user types. Use the Turbo Frame `data-turbo-action` attribute:

```erb
<%= form_with url: tasks_path,
      method: :get,
      data: {
        turbo_frame: "tasks_results",
        turbo_action: "advance",
        controller: "auto-submit"
      } do |f| %>
```

`data-turbo-action="advance"` tells Turbo to push the form's action URL (with query parameters) to the browser's history stack. The user can bookmark `https://app.com/tasks?q=groceries` and share it.

## Combining Search with Filters

When search lives alongside filters (status, category, date range), include all filter fields in the same form so they submit together:

```erb
<%= form_with url: tasks_path,
      method: :get,
      data: {
        turbo_frame: "tasks_results",
        controller: "auto-submit"
      } do |f| %>
  <div class="flex gap-4">
    <%= f.search_field :q,
          value: params[:q],
          placeholder: "Search...",
          data: { action: "input->auto-submit#submit" } %>

    <%= f.select :status,
          options_for_select([["All", ""], "Open", "Closed"], params[:status]),
          {},
          { data: { action: "change->auto-submit#submit" } } %>

    <%= f.select :category,
          options_for_select(Category.pluck(:name), params[:category]),
          { include_blank: "All Categories" },
          { data: { action: "change->auto-submit#submit" } } %>
  </div>
<% end %>
```

The text input triggers on `input` events (with debounce), while select fields trigger on `change` events (immediately).

## Pattern Card

### GOOD: Frame-Targeted Search with Debounce

```erb
<%= form_with url: tasks_path,
      method: :get,
      data: {
        turbo_frame: "tasks_results",
        turbo_action: "advance",
        controller: "auto-submit",
        auto_submit_delay_value: 300
      } do |f| %>
  <%= f.search_field :q,
        value: params[:q],
        placeholder: "Search tasks...",
        data: { action: "input->auto-submit#submit" },
        autocomplete: "off" %>
<% end %>

<%= turbo_frame_tag "tasks_results" do %>
  <%= render @tasks %>
<% end %>
```

```ruby
def index
  @tasks = Task.all
  @tasks = @tasks.search(params[:q]) if params[:q].present?
end
```

This approach uses a GET form for bookmarkable URLs, Turbo Frames for scoped result replacement, Stimulus for debounced auto-submit, CSS-only loading states via `aria-busy`, and `data-turbo-action="advance"` for browser history integration.

### BAD: Custom Fetch with innerHTML

```javascript
// Manual fetch, no Turbo, no debounce, no loading state
const searchInput = document.querySelector("#search")
searchInput.addEventListener("input", async (e) => {
  const response = await fetch(`/tasks/search?q=${e.target.value}`)
  const html = await response.text()
  document.querySelector("#results").innerHTML = html
})
```

This approach bypasses Turbo entirely, has no debouncing (fires on every keystroke), uses `innerHTML` which does not run scripts or update Turbo state, provides no loading indicator, breaks browser back/forward navigation, produces URLs that are not bookmarkable, and requires manual CSRF handling for any POST forms in the results.
