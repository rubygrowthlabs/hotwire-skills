---
title: "Optimistic Updates with Morphing"
categories:
  - ux
  - turbo
  - real-time
tags:
  - optimistic-ui
  - turbo-8-morphing
  - stimulus
  - starting-style
  - color-mix
  - perceived-performance
description: >-
  Show expected state immediately, let Turbo 8 morph correct if needed. Covers
  Stimulus controllers for optimistic DOM updates, server broadcast reconciliation,
  @starting-style for enter animations on morphed elements, and color-mix() for
  hover/active state generation.
---

# Optimistic Updates with Morphing

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Optimistic DOM Update with Stimulus](#optimistic-dom-update-with-stimulus)
  - [Server Reconciliation via Turbo Stream Morph](#server-reconciliation-via-turbo-stream-morph)
  - [Handling Conflicts Between Optimistic and Server State](#handling-conflicts-between-optimistic-and-server-state)
  - [@starting-style for Enter Animations on Morphed Elements](#starting-style-for-enter-animations-on-morphed-elements)
  - [color-mix() for Interactive State Generation](#color-mix-for-interactive-state-generation)
  - [Optimistic Toggle Pattern](#optimistic-toggle-pattern)
- [Pattern Card](#pattern-card)

## Overview

Optimistic updates show the expected result of a user action immediately, before the server confirms it. The user clicks "Like" and the heart fills instantly. The server processes the request in the background. When the server responds, Turbo 8's morphing reconciles the DOM with the authoritative server state.

This pattern creates the perception of instant response. The key insight is that most user actions succeed -- optimistic updates are correct 99% of the time. For the 1% that fail, the morph smoothly corrects the state without a jarring page refresh.

The combination of Stimulus (for the optimistic DOM change), Turbo Streams with morph (for server reconciliation), `@starting-style` (for enter animations on new or changed elements), and `color-mix()` (for generating interactive states without extra variables) creates a complete system for fast, visually polished interactions.

## Implementation

### Optimistic DOM Update with Stimulus

The Stimulus controller handles the immediate DOM update when the user acts. It modifies the UI before the form submission reaches the server.

```javascript
// app/javascript/controllers/optimistic_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["count", "icon"]
  static values = {
    current: Number,
    toggled: { type: Boolean, default: false }
  }

  toggle() {
    // Immediately update the UI
    this.toggledValue = !this.toggledValue
    this.currentValue += this.toggledValue ? 1 : -1

    // Update visual state
    this.element.classList.toggle("is-active", this.toggledValue)
    this.countTarget.textContent = this.currentValue

    // Animate the icon
    this.iconTarget.classList.add("optimistic-pulse")
    this.iconTarget.addEventListener("animationend", () => {
      this.iconTarget.classList.remove("optimistic-pulse")
    }, { once: true })
  }
}
```

```erb
<%# app/views/posts/_like_button.html.erb %>
<%= button_to post_likes_path(@post),
  method: @post.liked_by?(current_user) ? :delete : :post,
  class: "like-button #{@post.liked_by?(current_user) ? 'is-active' : ''}",
  data: {
    controller: "optimistic",
    optimistic_current_value: @post.likes_count,
    optimistic_toggled_value: @post.liked_by?(current_user),
    action: "optimistic#toggle"
  } do %>
  <span data-optimistic-target="icon" class="like-icon">
    <svg><!-- heart icon --></svg>
  </span>
  <span data-optimistic-target="count"><%= @post.likes_count %></span>
<% end %>
```

```css
@layer components {
  .like-button {
    display: inline-flex;
    align-items: center;
    gap: var(--space-xs);
    padding: var(--space-xs) var(--space-sm);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-lg);
    background: var(--color-surface);
    color: var(--color-text-muted);
    cursor: pointer;
    transition: color 0.15s, border-color 0.15s;

    &.is-active {
      color: var(--color-danger);
      border-color: color-mix(in oklch, var(--color-danger), transparent 70%);
      background: color-mix(in oklch, var(--color-danger), transparent 95%);
    }

    &:hover {
      border-color: var(--color-danger);
    }
  }

  .like-icon svg {
    inline-size: 1.25rem;
    block-size: 1.25rem;
    transition: scale 0.15s ease-out;
  }

  .is-active .like-icon svg {
    fill: currentColor;
  }

  /* Pulse animation on optimistic toggle */
  .optimistic-pulse {
    animation: pulse-scale 0.3s ease-out;
  }

  @keyframes pulse-scale {
    0% { scale: 1; }
    50% { scale: 1.3; }
    100% { scale: 1; }
  }

  @media (prefers-reduced-motion: reduce) {
    .optimistic-pulse {
      animation: none;
    }
  }
}
```

### Server Reconciliation via Turbo Stream Morph

After the optimistic update, the server processes the action and broadcasts the correct state. Turbo 8 morphing reconciles the DOM without replacing the entire element.

```ruby
# app/controllers/likes_controller.rb
class LikesController < ApplicationController
  def create
    @post = Post.find(params[:post_id])
    @post.likes.create!(user: Current.user)

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: turbo_stream.morph(
          dom_id(@post, :like_button),
          partial: "posts/like_button",
          locals: { post: @post }
        )
      end
      format.html { redirect_to @post }
    end
  end

  def destroy
    @post = Post.find(params[:post_id])
    @post.likes.find_by!(user: Current.user).destroy!

    respond_to do |format|
      format.turbo_stream do
        render turbo_stream: turbo_stream.morph(
          dom_id(@post, :like_button),
          partial: "posts/like_button",
          locals: { post: @post }
        )
      end
      format.html { redirect_to @post }
    end
  end
end
```

The morph action compares the server-rendered HTML with the current DOM and applies only the differences. If the optimistic state matches the server state (the common case), the morph is a no-op. If it differs, the morph smoothly corrects it.

### Handling Conflicts Between Optimistic and Server State

When the optimistic update is wrong (race condition, validation failure, permission error), the morph corrects it. Add visual feedback for corrections so the user understands what happened.

```javascript
// app/javascript/controllers/optimistic_controller.js
// Extended with rollback awareness

export default class extends Controller {
  static targets = ["count", "icon"]
  static values = {
    current: Number,
    toggled: { type: Boolean, default: false },
    pendingAction: { type: Boolean, default: false }
  }

  toggle() {
    if (this.pendingActionValue) return // Prevent double-toggle

    this.pendingActionValue = true
    this.toggledValue = !this.toggledValue
    this.currentValue += this.toggledValue ? 1 : -1

    this.element.classList.toggle("is-active", this.toggledValue)
    this.element.classList.add("is-pending")
    this.countTarget.textContent = this.currentValue
  }

  // Called when Turbo morph updates this element
  // Morph preserves the controller instance and calls valueChanged
  currentValueChanged() {
    this.countTarget.textContent = this.currentValue
    this.element.classList.remove("is-pending")
    this.pendingActionValue = false
  }

  toggledValueChanged() {
    this.element.classList.toggle("is-active", this.toggledValue)

    // If the server state differs from what we optimistically set,
    // add a correction animation
    if (!this.pendingActionValue) {
      this.element.classList.add("state-corrected")
      this.element.addEventListener("animationend", () => {
        this.element.classList.remove("state-corrected")
      }, { once: true })
    }
  }
}
```

```css
@layer components {
  /* Subtle indicator that an action is pending */
  .like-button.is-pending {
    opacity: 0.8;
  }

  /* Brief shake when server corrects the optimistic state */
  .state-corrected {
    animation: correction-shake 0.3s ease-out;
  }

  @keyframes correction-shake {
    0%, 100% { translate: 0; }
    25% { translate: -2px 0; }
    75% { translate: 2px 0; }
  }

  @media (prefers-reduced-motion: reduce) {
    .state-corrected {
      animation: none;
      outline: 2px solid var(--color-warning);
      outline-offset: 2px;
    }
  }
}
```

### @starting-style for Enter Animations on Morphed Elements

When Turbo morph inserts new elements or changes element visibility, `@starting-style` provides entry animations. This CSS feature defines the initial state that elements transition from when they first appear in the DOM.

```css
@layer components {
  /* New items added to a list animate in */
  .list-item {
    transition: opacity 0.3s ease-out, translate 0.3s ease-out;

    @starting-style {
      opacity: 0;
      translate: 0 0.5rem;
    }
  }

  /* Toast notification slides in from the bottom */
  .toast {
    position: fixed;
    inset-block-end: var(--space-lg);
    inset-inline-end: var(--space-lg);
    padding: var(--space-md) var(--space-lg);
    background: var(--color-text);
    color: var(--color-surface);
    border-radius: var(--radius-md);
    box-shadow: var(--shadow-lg);
    transition: translate 0.3s ease-out, opacity 0.3s ease-out;

    @starting-style {
      translate: 0 1rem;
      opacity: 0;
    }
  }

  /* Badge count update pulses when morphed */
  .notification-count {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-inline-size: 1.5rem;
    block-size: 1.5rem;
    border-radius: var(--radius-lg);
    background: var(--color-danger);
    color: white;
    font-size: var(--text-sm);
    font-weight: 600;
    transition: scale 0.2s ease-out;

    @starting-style {
      scale: 0;
    }
  }

  /* Morphed content areas fade in */
  .morph-content {
    transition: opacity 0.2s ease-out;

    @starting-style {
      opacity: 0;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .list-item,
    .toast,
    .notification-count,
    .morph-content {
      transition: none;
    }
  }
}
```

`@starting-style` works with Turbo morph because morphing inserts new DOM nodes. The browser sees the new node, applies the `@starting-style` values as the initial state, and then transitions to the normal state. This gives morphed content smooth entry animations with zero JavaScript.

### color-mix() for Interactive State Generation

Use `color-mix()` to derive hover, active, focus, and disabled states from a single base color. This eliminates the need for separate color variables for every state.

```css
@layer components {
  .btn {
    --btn-color: var(--color-primary);
    background: var(--btn-color);
    color: white;
    padding: var(--space-sm) var(--space-md);
    border: none;
    border-radius: var(--radius-md);
    font-weight: 600;
    cursor: pointer;
    transition: background-color 0.15s;

    &:hover {
      background: color-mix(in oklch, var(--btn-color), white 15%);
    }

    &:active {
      background: color-mix(in oklch, var(--btn-color), black 10%);
    }

    &:focus-visible {
      outline: 2px solid var(--btn-color);
      outline-offset: 2px;
    }

    &:disabled {
      background: color-mix(in oklch, var(--btn-color), var(--color-surface) 50%);
      cursor: not-allowed;
    }

    /* Variant: danger */
    &.btn--danger {
      --btn-color: var(--color-danger);
    }

    /* Variant: success */
    &.btn--success {
      --btn-color: var(--color-success);
    }

    /* Variant: ghost */
    &.btn--ghost {
      background: transparent;
      color: var(--btn-color);

      &:hover {
        background: color-mix(in oklch, var(--btn-color), transparent 90%);
      }

      &:active {
        background: color-mix(in oklch, var(--btn-color), transparent 80%);
      }
    }
  }

  /* Interactive card with derived hover state */
  .interactive-card {
    --card-accent: var(--color-primary);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    padding: var(--space-lg);
    transition: border-color 0.15s, box-shadow 0.15s;

    &:hover {
      border-color: color-mix(in oklch, var(--card-accent), transparent 50%);
      box-shadow: 0 0 0 3px color-mix(in oklch, var(--card-accent), transparent 85%);
    }

    &.interactive-card--selected {
      border-color: var(--card-accent);
      background: color-mix(in oklch, var(--card-accent), transparent 95%);
    }
  }
}
```

The `color-mix()` function works in OKLCH color space, which produces perceptually uniform results. Mixing 15% white into a blue and 15% white into a red produces equivalently lighter results, unlike RGB mixing where the perceptual change varies by hue.

### Optimistic Toggle Pattern

A complete optimistic toggle pattern for features like bookmark, favorite, archive, and pin actions:

```javascript
// app/javascript/controllers/optimistic_toggle_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { active: Boolean }
  static classes = ["active", "pending"]

  toggle() {
    this.activeValue = !this.activeValue
    this.element.classList.add(...this.pendingClasses)
  }

  activeValueChanged() {
    this.element.classList.toggle(this.activeClass, this.activeValue)
    this.element.classList.remove(...this.pendingClasses)
  }
}
```

```erb
<%# Generic optimistic toggle partial %>
<%= button_to toggle_path,
  method: active ? :delete : :post,
  class: "toggle-action #{active ? 'is-active' : ''}",
  data: {
    controller: "optimistic-toggle",
    optimistic_toggle_active_value: active,
    optimistic_toggle_active_class: "is-active",
    optimistic_toggle_pending_class: "is-pending",
    action: "optimistic-toggle#toggle"
  } do %>
  <%= yield %>
<% end %>
```

```css
@layer components {
  .toggle-action {
    transition: color 0.15s, background-color 0.15s;

    &.is-active {
      color: var(--color-primary);
    }

    &.is-pending {
      opacity: 0.7;
    }
  }
}
```

## Pattern Card

### GOOD: Optimistic Update Reconciled by Morph

```erb
<%= button_to post_likes_path(@post),
  method: :post,
  id: dom_id(@post, :like_button),
  class: "like-button",
  data: {
    controller: "optimistic",
    optimistic_current_value: @post.likes_count,
    action: "optimistic#toggle"
  } do %>
  <span data-optimistic-target="icon">Heart</span>
  <span data-optimistic-target="count"><%= @post.likes_count %></span>
<% end %>
```

```css
@layer components {
  .like-button {
    transition: color 0.15s;

    &.is-active {
      color: var(--color-danger);
      background: color-mix(in oklch, var(--color-danger), transparent 95%);
    }
  }

  .optimistic-pulse {
    animation: pulse-scale 0.3s ease-out;
  }
}
```

The heart fills immediately on click. The count increments instantly. The server processes the like, then morphs the button with the authoritative state. If the optimistic state was correct (the usual case), the morph is invisible. If it was wrong, the morph corrects it with a subtle animation. The user never waits for a server round-trip to see their action reflected.

### BAD: Blocking UI Until Server Responds

```erb
<%= button_to post_likes_path(@post), method: :post, class: "like-button" do %>
  <span>Heart</span>
  <span><%= @post.likes_count %></span>
<% end %>
```

Without optimistic updates, the user clicks "Like" and sees nothing happen for 200-500ms while the request round-trips to the server. The button does not change, the count does not update, and there is no feedback that the click registered. On slow connections, this delay stretches to seconds, during which the user may click again (causing a double-like) or assume the button is broken. Even on fast connections, the 200ms gap is perceptible and makes the interface feel sluggish compared to native apps where taps feel instant.
