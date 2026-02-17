---
title: "Loading Indicators"
categories:
  - ux
  - turbo
  - performance
tags:
  - loading-state
  - progress-bar
  - turbo-drive
  - form-submission
  - perceived-performance
description: >-
  Providing visual feedback during Turbo Drive navigation and form submissions.
  Covers progress bar customization, form submission indicators with turbo:submit-start/end,
  data-turbo-submits-with for button text changes, and the 200ms delay pattern.
---

# Loading Indicators

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Turbo Drive Progress Bar](#turbo-drive-progress-bar)
  - [Custom CSS Progress Bar](#custom-css-progress-bar)
  - [Form Submission Indicators](#form-submission-indicators)
  - [Button Text Changes with data-turbo-submits-with](#button-text-changes-with-data-turbo-submits-with)
  - [Delayed Indicators: The 200ms Rule](#delayed-indicators-the-200ms-rule)
  - [Full-Page Loading Overlay](#full-page-loading-overlay)
- [Pattern Card](#pattern-card)

## Overview

Loading indicators bridge the gap between user action and system response. Without them, users cannot tell whether their click registered, whether the system is processing, or whether something went wrong. Good loading indicators follow three principles:

1. **Delay before showing.** Fast responses (under 200ms) should feel instant. Showing a spinner for 50ms creates a worse experience than showing nothing.
2. **Show progress, not just activity.** A progress bar moving forward is better than a spinner because it sets expectations about duration.
3. **Match the scope.** A full-page navigation gets a progress bar. A form submission gets a button state change. A frame load gets a localized spinner. The indicator should match the scope of the operation.

Turbo Drive includes a built-in progress bar that appears during navigations. For form submissions and frame loading, you need custom indicators.

## Implementation

### Turbo Drive Progress Bar

Turbo Drive automatically shows a thin progress bar at the top of the viewport when navigation takes longer than a configurable delay (default: 500ms). You can customize its appearance and timing.

```css
@layer components {
  /* The progress bar is a <div> with class .turbo-progress-bar */
  .turbo-progress-bar {
    background: var(--color-primary);
    block-size: 3px;

    /* Add a glow effect for visibility */
    box-shadow: 0 0 8px color-mix(in oklch, var(--color-primary), transparent 50%);
  }
}
```

Configure the delay before the progress bar appears:

```javascript
// app/javascript/application.js
import { Turbo } from "@hotwired/turbo-rails"

// Show progress bar after 200ms instead of the default 500ms
Turbo.setProgressBarDelay(200)
```

### Custom CSS Progress Bar

For more control, build a custom progress bar that replaces Turbo's default. This uses CSS animations to simulate progress.

```css
@layer components {
  .turbo-progress-bar {
    /* Hide the default Turbo bar */
    display: none;
  }

  .custom-progress {
    position: fixed;
    inset-block-start: 0;
    inset-inline-start: 0;
    inline-size: 0;
    block-size: 3px;
    background: linear-gradient(
      90deg,
      var(--color-primary),
      color-mix(in oklch, var(--color-primary), white 30%)
    );
    z-index: 9999;
    transition: inline-size 0.3s ease-out;
    pointer-events: none;

    /* Only show after 200ms delay */
    opacity: 0;
    animation: fadeIn 0.1s ease-in 0.2s forwards;

    &[data-state="loading"] {
      /* Animate to 80% quickly, then slow down */
      animation:
        fadeIn 0.1s ease-in 0.2s forwards,
        progress-indeterminate 2s ease-out 0.2s forwards;
    }

    &[data-state="complete"] {
      inline-size: 100%;
      opacity: 1;
      animation: progress-complete 0.3s ease-out forwards;
    }
  }

  @keyframes fadeIn {
    to { opacity: 1; }
  }

  @keyframes progress-indeterminate {
    0% { inline-size: 0; }
    50% { inline-size: 60%; }
    100% { inline-size: 85%; }
  }

  @keyframes progress-complete {
    0% { inline-size: 85%; opacity: 1; }
    100% { inline-size: 100%; opacity: 0; }
  }
}
```

```javascript
// app/javascript/controllers/progress_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["bar"]

  connect() {
    document.addEventListener("turbo:visit", this.start)
    document.addEventListener("turbo:load", this.complete)
    document.addEventListener("turbo:fetch-request-error", this.complete)
  }

  disconnect() {
    document.removeEventListener("turbo:visit", this.start)
    document.removeEventListener("turbo:load", this.complete)
    document.removeEventListener("turbo:fetch-request-error", this.complete)
  }

  start = () => {
    this.barTarget.dataset.state = "loading"
  }

  complete = () => {
    this.barTarget.dataset.state = "complete"
    // Reset after animation completes
    setTimeout(() => {
      this.barTarget.dataset.state = ""
    }, 300)
  }
}
```

### Form Submission Indicators

Use the `turbo:submit-start` and `turbo:submit-end` events to show loading state on forms. The form element receives an `aria-busy="true"` attribute automatically during submission.

```css
@layer components {
  /* Style the form during submission */
  form[aria-busy="true"] {
    pointer-events: none;

    & .form-actions {
      opacity: 0.6;
    }

    /* Show a spinner next to the submit button */
    & [type="submit"]::after {
      content: "";
      display: inline-block;
      inline-size: 1em;
      block-size: 1em;
      margin-inline-start: var(--space-sm);
      border: 2px solid currentColor;
      border-inline-end-color: transparent;
      border-radius: 50%;
      animation: spin 0.6s linear infinite;
      vertical-align: middle;
    }
  }

  @keyframes spin {
    to { rotate: 360deg; }
  }
}
```

For custom behavior beyond CSS, listen to the Turbo events:

```javascript
// app/javascript/controllers/form_loading_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["submit", "spinner"]

  connect() {
    this.element.addEventListener("turbo:submit-start", this.showLoading)
    this.element.addEventListener("turbo:submit-end", this.hideLoading)
  }

  disconnect() {
    this.element.removeEventListener("turbo:submit-start", this.showLoading)
    this.element.removeEventListener("turbo:submit-end", this.hideLoading)
  }

  showLoading = () => {
    this.submitTarget.disabled = true
    this.spinnerTarget.hidden = false
  }

  hideLoading = () => {
    this.submitTarget.disabled = false
    this.spinnerTarget.hidden = true
  }
}
```

### Button Text Changes with data-turbo-submits-with

Turbo provides a built-in attribute to change button text during form submission. No JavaScript required.

```erb
<%# The button text changes to "Saving..." during submission %>
<%= form_with model: @post do |f| %>
  <%= f.text_field :title %>
  <%= f.text_area :body %>
  <%= f.submit "Save Post", data: { turbo_submits_with: "Saving..." } %>
<% end %>
```

This is the simplest loading indicator pattern. The button text changes when the form submits and reverts when the response arrives. Combine it with CSS to also disable the button:

```css
@layer components {
  /* Buttons with turbo_submits_with get disabled attribute during submission */
  button[disabled],
  input[type="submit"][disabled] {
    opacity: 0.6;
    cursor: not-allowed;
  }
}
```

For more descriptive button states, use it with an icon:

```erb
<%= f.submit "Create Account",
  data: { turbo_submits_with: "Creating account..." },
  class: "btn btn--primary" %>
```

### Delayed Indicators: The 200ms Rule

The most important UX principle for loading indicators is the delay. Showing a spinner for 50ms before content appears creates a janky flash that makes the app feel slower than if no indicator appeared at all.

The 200ms threshold is based on human perception research: actions that complete within 200ms feel instant. Actions that take 200ms-1s need a subtle indicator. Actions over 1s need a clear progress signal.

```css
@layer components {
  /* Generic delayed loading indicator */
  .loading-indicator {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: var(--space-lg);

    /* Hidden for the first 200ms */
    opacity: 0;
    animation: loading-appear 0.2s ease-in 0.2s forwards;
  }

  @keyframes loading-appear {
    to { opacity: 1; }
  }

  /* Three-dot loading animation */
  .loading-dots {
    display: flex;
    gap: var(--space-xs);

    & span {
      inline-size: 0.5rem;
      block-size: 0.5rem;
      border-radius: 50%;
      background: var(--color-text-muted);
      animation: dot-bounce 1.4s ease-in-out infinite both;

      &:nth-child(1) { animation-delay: -0.32s; }
      &:nth-child(2) { animation-delay: -0.16s; }
    }
  }

  @keyframes dot-bounce {
    0%, 80%, 100% { scale: 0; }
    40% { scale: 1; }
  }

  /* Respect reduced motion */
  @media (prefers-reduced-motion: reduce) {
    .loading-dots span {
      animation: none;
      opacity: 0.4;

      &:nth-child(2) { opacity: 0.6; }
      &:nth-child(3) { opacity: 0.8; }
    }
  }
}
```

```erb
<div class="loading-indicator">
  <div class="loading-dots">
    <span></span>
    <span></span>
    <span></span>
  </div>
  <span class="sr-only">Loading...</span>
</div>
```

### Full-Page Loading Overlay

For destructive or long-running operations (bulk deletes, exports), use a full-page overlay that prevents interaction.

```css
@layer components {
  .page-loading-overlay {
    position: fixed;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: oklch(0% 0 0 / 0.3);
    z-index: 9999;
    backdrop-filter: blur(2px);

    /* 200ms delay before showing */
    opacity: 0;
    animation: fadeIn 0.2s ease-in 0.2s forwards;

    & .overlay-content {
      background: var(--color-surface);
      padding: var(--space-xl);
      border-radius: var(--radius-lg);
      box-shadow: var(--shadow-lg);
      text-align: center;

      & p {
        margin-block-start: var(--space-md);
        color: var(--color-text-muted);
      }
    }
  }
}
```

## Pattern Card

### GOOD: Delayed Loading Indicator with Button State Change

```erb
<%= form_with model: @project, data: { controller: "form-loading" } do |f| %>
  <%= f.text_field :name, required: true %>
  <%= f.text_area :description %>

  <div class="form-actions">
    <%= f.submit "Create Project",
      class: "btn btn--primary",
      data: { turbo_submits_with: "Creating..." } %>
  </div>
<% end %>
```

```css
@layer components {
  form[aria-busy="true"] .btn {
    opacity: 0.7;
    cursor: wait;
  }
}
```

The button text changes immediately to "Creating..." giving instant feedback. The `aria-busy` attribute styles the form to prevent double submission. If the server responds in under 200ms, the user sees "Create Project" change briefly to "Creating..." and then the response -- no spinner flash, no jarring transition.

### BAD: Instant Spinner Causing Flash on Fast Loads

```erb
<%= form_with model: @project do |f| %>
  <%= f.text_field :name %>
  <button type="submit" onclick="showSpinner()">
    <span class="btn-text">Create Project</span>
    <span class="spinner" style="display:none">
      <img src="/spinner.gif" alt="Loading">
    </span>
  </button>
<% end %>

<script>
function showSpinner() {
  document.querySelector('.btn-text').style.display = 'none';
  document.querySelector('.spinner').style.display = 'inline';
}
</script>
```

This pattern shows the spinner immediately on every submission. For the 90% of requests that complete in under 200ms, users see an annoying flash: button text disappears, spinner appears, spinner disappears, page changes. The spinner makes the app feel slower than it actually is. Additionally, the inline JavaScript does not clean up on Turbo navigation and the spinner image adds an unnecessary network request.
