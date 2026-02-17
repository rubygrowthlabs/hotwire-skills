---
title: Optimistic UI Patterns with Turbo
date: 2025-02-15
categories:
  - Turbo Streams
tags:
  - turbo-streams
  - optimistic-ui
  - stimulus
  - rollback
  - perceived-performance
description: Show expected results before server confirmation using Stimulus controllers that update the DOM optimistically and handle rollback on failure.
---

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Basic Optimistic Pattern](#basic-optimistic-pattern)
  - [Optimistic Like Button](#optimistic-like-button)
  - [Optimistic List Append](#optimistic-list-append)
  - [Handling Failures and Rollback](#handling-failures-and-rollback)
  - [Server Confirmation via Turbo Stream](#server-confirmation-via-turbo-stream)
  - [Optimistic Delete with Undo](#optimistic-delete-with-undo)
- [Key Points](#key-points)
- [Pattern Card: Optimistic Update with Rollback](#pattern-card-optimistic-update-with-rollback)

## Overview

Optimistic UI shows the expected result of a user action immediately, before the server confirms the change. The client updates the DOM as if the operation succeeded, then the server either confirms (no visible change) or corrects (rollback to actual state). This eliminates the perceived latency of network round trips and makes the interface feel instant.

The pattern works well for actions with a high success rate (likes, toggles, simple CRUD). It pairs naturally with Turbo Streams: the Stimulus controller handles the optimistic update, and the server sends a Turbo Stream response that either confirms or corrects the DOM.

## Implementation

### Basic Optimistic Pattern

The general pattern involves three steps:

1. **Stimulus controller** intercepts the action and immediately updates the DOM
2. **Form submits** to the server in the background via Turbo
3. **Server responds** with a Turbo Stream that either matches the optimistic state (no visible change) or corrects it (rollback)

### Optimistic Like Button

A like button that toggles instantly without waiting for the server:

```erb
<%# app/views/posts/_like_button.html.erb %>
<div id="<%= dom_id(post, :like) %>"
     data-controller="optimistic-toggle"
     data-optimistic-toggle-active-class="liked"
     data-optimistic-toggle-count-value="<%= post.likes_count %>">

  <%= button_to post_likes_path(post),
        method: post.liked_by?(Current.user) ? :delete : :post,
        class: "like-btn #{post.liked_by?(Current.user) ? 'liked' : ''}",
        data: {
          action: "optimistic-toggle#toggle",
          optimistic_toggle_target: "button"
        } do %>
    <svg class="heart-icon"><use href="#heart" /></svg>
    <span data-optimistic-toggle-target="count"><%= post.likes_count %></span>
  <% end %>
</div>
```

```javascript
// app/javascript/controllers/optimistic_toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["button", "count"]
  static classes = ["active"]
  static values = { count: Number }

  toggle(event) {
    // Save state for potential rollback
    this.previousCount = this.countValue
    this.wasActive = this.element.classList.contains(this.activeClass)

    // Optimistically toggle
    if (this.wasActive) {
      this.element.classList.remove(this.activeClass)
      this.countValue -= 1
    } else {
      this.element.classList.add(this.activeClass)
      this.countValue += 1
    }

    this.countTarget.textContent = this.countValue
  }

  // Called if the form submission fails
  rollback() {
    this.countValue = this.previousCount
    this.countTarget.textContent = this.countValue

    if (this.wasActive) {
      this.element.classList.add(this.activeClass)
    } else {
      this.element.classList.remove(this.activeClass)
    }
  }
}
```

The server responds with a Turbo Stream that renders the canonical state:

```erb
<%# app/views/likes/create.turbo_stream.erb %>
<%= turbo_stream.replace dom_id(@post, :like) do %>
  <%= render "posts/like_button", post: @post %>
<% end %>
```

If the server state matches the optimistic update, the user sees no flicker. If the server rejects the action (e.g., the user already liked the post), the stream replaces the optimistic DOM with the correct state.

### Optimistic List Append

Add an item to a list immediately while the server processes it:

```erb
<%# app/views/todos/_form.html.erb %>
<%= form_with model: Todo.new,
              url: todos_path,
              data: {
                controller: "optimistic-append",
                action: "turbo:submit-start->optimistic-append#appendOptimistic turbo:submit-end->optimistic-append#cleanup",
                optimistic_append_target_value: "todos"
              } do |f| %>
  <%= f.text_field :title,
        data: { optimistic_append_target: "input" },
        placeholder: "Add a todo..." %>
  <%= f.submit "Add" %>
<% end %>
```

```javascript
// app/javascript/controllers/optimistic_append_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input"]
  static values = { target: String }

  appendOptimistic(event) {
    const title = this.inputTarget.value.trim()
    if (!title) return

    // Create optimistic element
    const optimisticId = `optimistic-${Date.now()}`
    const container = document.getElementById(this.targetValue)
    const html = `
      <div id="${optimisticId}" class="todo-item todo-item--pending">
        <span class="todo-title">${this.escapeHtml(title)}</span>
        <span class="todo-status">Saving...</span>
      </div>
    `
    container.insertAdjacentHTML("beforeend", html)
    this.optimisticElement = document.getElementById(optimisticId)

    // Clear input
    this.inputTarget.value = ""
  }

  cleanup(event) {
    // Remove optimistic element after server responds
    // The server's turbo_stream.append will add the real element
    if (this.optimisticElement) {
      // Small delay to allow stream to arrive first
      setTimeout(() => {
        this.optimisticElement?.remove()
      }, 100)
    }
  }

  escapeHtml(text) {
    const div = document.createElement("div")
    div.textContent = text
    return div.innerHTML
  }
}
```

### Handling Failures and Rollback

Listen for Turbo fetch failures to trigger rollback:

```javascript
// app/javascript/controllers/optimistic_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { rollbackHtml: String }

  connect() {
    this.element.addEventListener("turbo:submit-end", this.handleResponse.bind(this))
  }

  saveState() {
    // Snapshot current DOM for rollback
    this.rollbackHtmlValue = this.element.innerHTML
  }

  handleResponse(event) {
    if (!event.detail.success) {
      this.rollback()
      this.showError()
    }
  }

  rollback() {
    if (this.rollbackHtmlValue) {
      this.element.innerHTML = this.rollbackHtmlValue
      this.rollbackHtmlValue = ""
    }
  }

  showError() {
    const flash = document.getElementById("flash_messages")
    if (flash) {
      flash.insertAdjacentHTML("beforeend",
        `<div class="flash flash--error" data-controller="auto-dismiss">
           Something went wrong. Please try again.
         </div>`
      )
    }
  }
}
```

For network errors, use `turbo:fetch-request-error`:

```javascript
document.addEventListener("turbo:fetch-request-error", (event) => {
  // Handle network failures globally
  const target = event.target.closest("[data-controller*='optimistic']")
  if (target) {
    const controller = target.__stimulusControllers?.find(
      c => c.identifier.includes("optimistic")
    )
    controller?.rollback()
  }
})
```

### Server Confirmation via Turbo Stream

The server always sends the authoritative state. Design your stream templates to render the canonical markup:

```ruby
# app/controllers/todos_controller.rb
class TodosController < ApplicationController
  def create
    @todo = Current.user.todos.build(todo_params)

    if @todo.save
      respond_to do |format|
        format.turbo_stream  # renders create.turbo_stream.erb
        format.html { redirect_to todos_path }
      end
    else
      respond_to do |format|
        format.turbo_stream do
          render turbo_stream: turbo_stream.replace(
            "new_todo_form",
            partial: "todos/form",
            locals: { todo: @todo }
          )
        end
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end
end
```

```erb
<%# app/views/todos/create.turbo_stream.erb %>
<%= turbo_stream.append "todos", partial: "todos/todo", locals: { todo: @todo } %>

<%# Reset the form %>
<%= turbo_stream.replace "new_todo_form" do %>
  <%= render "todos/form", todo: Todo.new %>
<% end %>
```

### Optimistic Delete with Undo

Hide the element immediately and provide an undo window:

```javascript
// app/javascript/controllers/optimistic_delete_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { undoDuration: { type: Number, default: 5000 } }

  delete(event) {
    event.preventDefault()

    // Hide the element optimistically
    this.element.style.opacity = "0"
    this.element.style.transition = "opacity 0.3s ease"

    // Show undo toast
    this.undoTimeout = setTimeout(() => {
      // Actually submit the delete form
      this.element.querySelector("form[method='post'] input[name='_method'][value='delete']")
        ?.closest("form")
        ?.requestSubmit()
    }, this.undoDurationValue)

    this.showUndoToast()
  }

  undo() {
    clearTimeout(this.undoTimeout)
    this.element.style.opacity = "1"
    this.dismissUndoToast()
  }

  showUndoToast() {
    const toast = document.createElement("div")
    toast.id = "undo-toast"
    toast.className = "toast toast--undo"
    toast.innerHTML = `
      Item will be deleted.
      <button data-action="optimistic-delete#undo">Undo</button>
    `
    document.getElementById("flash_messages")?.appendChild(toast)
  }

  dismissUndoToast() {
    document.getElementById("undo-toast")?.remove()
  }
}
```

## Key Points

- Optimistic UI immediately reflects expected state, then the server confirms or corrects
- Use Stimulus controllers to manage the optimistic DOM update and snapshot state for rollback
- The server always sends the authoritative state via Turbo Stream
- If the optimistic and server states match, the user sees no change (seamless)
- If they differ, the Turbo Stream replaces the optimistic DOM with the correct state
- Listen to `turbo:submit-end` and check `event.detail.success` to trigger rollback
- Best suited for high-success-rate actions: likes, toggles, simple creates
- Always escape user-generated content when building optimistic HTML

## Pattern Card: Optimistic Update with Rollback

**When to use**: User actions with high success rates (likes, toggles, simple CRUD) where eliminating perceived latency improves the experience.

**GOOD - Optimistic update with server confirmation and rollback**:

```javascript
// Stimulus controller toggles immediately
toggle(event) {
  this.previousState = this.element.classList.contains("active")
  this.element.classList.toggle("active")
  this.countTarget.textContent = this.previousState
    ? parseInt(this.countTarget.textContent) - 1
    : parseInt(this.countTarget.textContent) + 1
}

// Server confirms via Turbo Stream (user sees nothing if states match)
// Or server corrects by replacing with canonical markup
```

```erb
<%# Server always renders the truth %>
<%= turbo_stream.replace dom_id(@post, :like) do %>
  <%= render "posts/like_button", post: @post %>
<% end %>
```

**BAD - Blocking the UI waiting for the server to respond**:

```erb
<%# Button shows spinner, user waits 200-500ms for every toggle %>
<%= button_to post_likes_path(@post),
      method: :post,
      class: "like-btn",
      data: { turbo_submits_with: "..." } do %>
  <svg class="heart-icon"><use href="#heart" /></svg>
  <span><%= @post.likes_count %></span>
<% end %>
```

The user clicks, sees a spinner, waits for the network round trip, then sees the result. For a simple toggle this feels sluggish.
