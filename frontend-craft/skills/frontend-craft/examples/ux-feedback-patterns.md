---
title: "UX Feedback Patterns"
categories:
  - ux
  - examples
  - patterns
tags:
  - optimistic-update
  - toast-notification
  - view-transitions
  - form-submission
  - skeleton-screen
  - stimulus
description: >-
  Complete UX feedback patterns combining multiple techniques: like button with
  optimistic count, toast notifications with CSS slide-in, page transitions with
  view-transition-name, form submission with button state change, and lazy-loaded
  dashboard cards with skeleton placeholders.
---

# UX Feedback Patterns

Complete, copy-ready examples combining CSS, HTML, and Stimulus patterns from the frontend-craft skill. Each example is self-contained and can be dropped into a Rails application with Turbo and Stimulus.

## Table of Contents

- [Example 1: Like Button with Optimistic Count Update](#example-1-like-button-with-optimistic-count-update)
- [Example 2: Toast Notifications with CSS-Only Slide-In](#example-2-toast-notifications-with-css-only-slide-in)
- [Example 3: Page Transition with view-transition-name Persistence](#example-3-page-transition-with-view-transition-name-persistence)
- [Example 4: Form Submission with Button State and Success Animation](#example-4-form-submission-with-button-state-and-success-animation)
- [Example 5: Lazy-Loaded Dashboard Card with Skeleton Placeholder](#example-5-lazy-loaded-dashboard-card-with-skeleton-placeholder)

---

## Example 1: Like Button with Optimistic Count Update

A like button that updates instantly on click, with a heart animation and count increment. The server reconciles via Turbo Stream morph.

### Stimulus Controller

```javascript
// app/javascript/controllers/like_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["count", "icon"]
  static values = {
    count: Number,
    liked: Boolean
  }

  toggle() {
    // Optimistic update
    this.likedValue = !this.likedValue
    this.countValue += this.likedValue ? 1 : -1
  }

  likedValueChanged() {
    this.element.classList.toggle("like-button--active", this.likedValue)
    this.iconTarget.classList.add("like-icon--animating")
    this.iconTarget.addEventListener("animationend", () => {
      this.iconTarget.classList.remove("like-icon--animating")
    }, { once: true })
  }

  countValueChanged() {
    this.countTarget.textContent = this.countValue
  }
}
```

### HTML (ERB)

```erb
<%# app/views/posts/_like_button.html.erb %>
<div id="<%= dom_id(post, :like_button) %>">
  <%= button_to post_likes_path(post),
    method: post.liked_by?(current_user) ? :delete : :post,
    class: "like-button #{'like-button--active' if post.liked_by?(current_user)}",
    data: {
      controller: "like",
      like_count_value: post.likes_count,
      like_liked_value: post.liked_by?(current_user),
      action: "like#toggle"
    } do %>
    <span data-like-target="icon" class="like-icon">
      <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none"
           stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"/>
      </svg>
    </span>
    <span data-like-target="count" class="like-count"><%= post.likes_count %></span>
  <% end %>
</div>
```

### CSS

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
    font-size: var(--text-sm);
    cursor: pointer;
    transition: color 0.15s, border-color 0.15s, background-color 0.15s;
    user-select: none;

    &:hover {
      color: oklch(55% 0.22 25);
      border-color: color-mix(in oklch, oklch(55% 0.22 25), transparent 70%);
    }

    &.like-button--active {
      color: oklch(55% 0.22 25);
      background: color-mix(in oklch, oklch(55% 0.22 25), transparent 95%);
      border-color: color-mix(in oklch, oklch(55% 0.22 25), transparent 70%);

      & .like-icon svg {
        fill: currentColor;
      }
    }
  }

  .like-icon {
    display: flex;

    & svg {
      inline-size: 1.125rem;
      block-size: 1.125rem;
      transition: fill 0.15s;
    }
  }

  .like-icon--animating {
    animation: like-bounce 0.35s ease-out;
  }

  @keyframes like-bounce {
    0% { scale: 1; }
    30% { scale: 1.35; }
    60% { scale: 0.9; }
    100% { scale: 1; }
  }

  .like-count {
    font-weight: 600;
    min-inline-size: 1ch;
    text-align: center;
  }

  @media (prefers-reduced-motion: reduce) {
    .like-icon--animating { animation: none; }
  }
}
```

### Server Response (Turbo Stream)

```erb
<%# app/views/likes/create.turbo_stream.erb %>
<%= turbo_stream.morph dom_id(@post, :like_button) do %>
  <%= render "posts/like_button", post: @post %>
<% end %>
```

---

## Example 2: Toast Notifications with CSS-Only Slide-In

Toast notifications that slide in from the bottom-right, auto-dismiss after a delay, and support manual dismissal. Pure CSS animation with a Stimulus controller only for auto-dismiss timing.

### Stimulus Controller

```javascript
// app/javascript/controllers/toast_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    duration: { type: Number, default: 5000 }
  }

  connect() {
    this.timeout = setTimeout(() => this.dismiss(), this.durationValue)
  }

  disconnect() {
    clearTimeout(this.timeout)
  }

  dismiss() {
    this.element.classList.add("toast--dismissing")
    this.element.addEventListener("animationend", () => {
      this.element.remove()
    }, { once: true })
  }
}
```

### HTML (ERB)

```erb
<%# app/views/shared/_toasts.html.erb %>
<div id="toast-container" class="toast-container" aria-live="polite" aria-atomic="false">
  <% flash.each do |type, message| %>
    <div class="toast toast--<%= type %>"
         role="status"
         data-controller="toast"
         data-toast-duration-value="5000">
      <div class="toast-content">
        <span class="toast-icon">
          <% case type.to_s %>
          <% when "notice", "success" %>
            <svg viewBox="0 0 20 20" fill="currentColor"><path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z"/></svg>
          <% when "alert", "error" %>
            <svg viewBox="0 0 20 20" fill="currentColor"><path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7 4a1 1 0 11-2 0 1 1 0 012 0zm-1-9a1 1 0 00-1 1v4a1 1 0 102 0V6a1 1 0 00-1-1z"/></svg>
          <% end %>
        </span>
        <p class="toast-message"><%= message %></p>
      </div>
      <button class="toast-close" data-action="toast#dismiss" aria-label="Dismiss">
        <svg viewBox="0 0 20 20" fill="currentColor"><path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z"/></svg>
      </button>
    </div>
  <% end %>
</div>
```

### CSS

```css
@layer components {
  .toast-container {
    position: fixed;
    inset-block-end: var(--space-lg);
    inset-inline-end: var(--space-lg);
    display: flex;
    flex-direction: column-reverse;
    gap: var(--space-sm);
    z-index: 9999;
    pointer-events: none;

    /* Stack from bottom */
    & > * {
      pointer-events: auto;
    }
  }

  .toast {
    display: flex;
    align-items: flex-start;
    gap: var(--space-sm);
    padding: var(--space-md) var(--space-lg);
    background: var(--color-text);
    color: var(--color-surface);
    border-radius: var(--radius-md);
    box-shadow: var(--shadow-lg);
    max-inline-size: 24rem;

    /* Entry animation */
    transition: translate 0.3s ease-out, opacity 0.3s ease-out;

    @starting-style {
      translate: 0 1rem;
      opacity: 0;
    }

    /* Auto-dismiss progress bar at the bottom */
    &::after {
      content: "";
      position: absolute;
      inset-block-end: 0;
      inset-inline-start: 0;
      block-size: 3px;
      background: color-mix(in oklch, white, transparent 50%);
      border-radius: 0 0 var(--radius-md) var(--radius-md);
      animation: toast-timer 5s linear forwards;
    }

    /* Variants */
    &.toast--notice,
    &.toast--success {
      background: oklch(40% 0.15 145);
    }

    &.toast--alert,
    &.toast--error {
      background: oklch(40% 0.18 25);
    }
  }

  /* Dismiss animation */
  .toast--dismissing {
    animation: toast-dismiss 0.2s ease-in forwards;
  }

  @keyframes toast-timer {
    from { inline-size: 100%; }
    to { inline-size: 0%; }
  }

  @keyframes toast-dismiss {
    to {
      translate: 1rem 0;
      opacity: 0;
    }
  }

  .toast-content {
    display: flex;
    align-items: flex-start;
    gap: var(--space-sm);
    flex: 1;
  }

  .toast-icon {
    flex-shrink: 0;

    & svg {
      inline-size: 1.25rem;
      block-size: 1.25rem;
    }
  }

  .toast-message {
    font-size: var(--text-sm);
    line-height: 1.4;
  }

  .toast-close {
    flex-shrink: 0;
    background: none;
    border: none;
    color: inherit;
    opacity: 0.6;
    cursor: pointer;
    padding: 0;

    & svg {
      inline-size: 1rem;
      block-size: 1rem;
    }

    &:hover {
      opacity: 1;
    }
  }

  @media (prefers-reduced-motion: reduce) {
    .toast {
      transition: none;
    }

    .toast--dismissing {
      animation: none;
      display: none;
    }

    .toast::after {
      animation: none;
    }
  }

  /* Mobile: full-width toasts at bottom */
  @media (width < 640px) {
    .toast-container {
      inset-inline-start: var(--space-md);
      inset-inline-end: var(--space-md);
    }

    .toast {
      max-inline-size: none;
    }
  }
}
```

### Turbo Stream for Server-Pushed Toasts

```erb
<%# Append a toast from a Turbo Stream response %>
<%= turbo_stream.append "toast-container" do %>
  <div class="toast toast--success"
       role="status"
       data-controller="toast"
       data-toast-duration-value="5000">
    <div class="toast-content">
      <p class="toast-message">Project saved successfully.</p>
    </div>
    <button class="toast-close" data-action="toast#dismiss" aria-label="Dismiss">
      <svg viewBox="0 0 20 20" fill="currentColor"><path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z"/></svg>
    </button>
  </div>
<% end %>
```

---

## Example 3: Page Transition with view-transition-name Persistence

A project list where clicking a card transitions smoothly into the detail page. The card morphs into the detail view header using the View Transitions API.

### Index Page

```erb
<%# app/views/projects/index.html.erb %>
<h1>Projects</h1>

<div class="projects-grid">
  <% @projects.each do |project| %>
    <%= link_to project_path(project), class: "project-card",
      style: "view-transition-name: project-#{project.id}" do %>
      <div class="project-card-color"
           style="background: <%= project.color %>"></div>
      <h3 class="project-card-title"><%= project.name %></h3>
      <p class="project-card-description"><%= truncate(project.description, length: 100) %></p>
      <div class="project-card-meta">
        <span><%= project.tasks_count %> tasks</span>
        <span><%= time_ago_in_words(project.updated_at) %> ago</span>
      </div>
    <% end %>
  <% end %>
</div>
```

### Detail Page

```erb
<%# app/views/projects/show.html.erb %>
<article class="project-detail"
         style="view-transition-name: project-<%= @project.id %>">
  <div class="project-detail-header"
       style="background: <%= @project.color %>">
    <h1><%= @project.name %></h1>
  </div>
  <div class="project-detail-body">
    <p><%= @project.description %></p>
    <%# ... rest of project detail ... %>
  </div>
</article>
```

### Layout

```erb
<%# app/views/layouts/application.html.erb %>
<head>
  <meta name="view-transition" content="same-origin">
  <%# ... other head tags ... %>
</head>
```

### CSS

```css
@layer components {
  .projects-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: var(--space-lg);
  }

  .project-card {
    display: block;
    text-decoration: none;
    color: inherit;
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    overflow: hidden;
    transition: box-shadow 0.15s, translate 0.15s;

    &:hover {
      box-shadow: var(--shadow-md);
      translate: 0 -2px;
    }
  }

  .project-card-color {
    block-size: 4px;
  }

  .project-card-title {
    font-size: var(--text-lg);
    font-weight: 600;
    padding: var(--space-md) var(--space-md) 0;
  }

  .project-card-description {
    padding: var(--space-sm) var(--space-md);
    color: var(--color-text-muted);
    font-size: var(--text-sm);
  }

  .project-card-meta {
    display: flex;
    justify-content: space-between;
    padding: var(--space-sm) var(--space-md) var(--space-md);
    font-size: var(--text-sm);
    color: var(--color-text-muted);
  }

  .project-detail {
    border-radius: var(--radius-lg);
    overflow: hidden;
    border: 1px solid var(--color-border);
  }

  .project-detail-header {
    padding: var(--space-2xl) var(--space-xl);
    color: white;

    & h1 {
      font-size: var(--text-2xl);
    }
  }

  .project-detail-body {
    padding: var(--space-xl);
  }

  /* View Transition animations */
  @supports (view-transition-name: none) {
    ::view-transition-old(main-content) {
      animation: fade-and-shrink 0.2s ease-out forwards;
    }

    ::view-transition-new(main-content) {
      animation: grow-and-fade-in 0.2s ease-out forwards;
    }

    @keyframes fade-and-shrink {
      to { opacity: 0; scale: 0.98; }
    }

    @keyframes grow-and-fade-in {
      from { opacity: 0; scale: 1.02; }
      to { opacity: 1; scale: 1; }
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

---

## Example 4: Form Submission with Button State Change and Success Animation

A form that disables the button during submission, shows a brief success animation when complete, then redirects or updates.

### HTML (ERB)

```erb
<%# app/views/projects/_form.html.erb %>
<%= form_with model: project,
  class: "project-form",
  data: { controller: "form-feedback" } do |f| %>

  <div class="form-group">
    <%= f.label :name, class: "form-label" %>
    <%= f.text_field :name, class: "form-input", required: true %>
  </div>

  <div class="form-group">
    <%= f.label :description, class: "form-label" %>
    <%= f.text_area :description, class: "form-input", rows: 4 %>
  </div>

  <div class="form-actions">
    <%= f.submit project.persisted? ? "Save Changes" : "Create Project",
      class: "btn btn--primary",
      data: {
        turbo_submits_with: project.persisted? ? "Saving..." : "Creating...",
        form_feedback_target: "submit"
      } %>
  </div>
<% end %>
```

### Stimulus Controller

```javascript
// app/javascript/controllers/form_feedback_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["submit"]

  connect() {
    this.element.addEventListener("turbo:submit-start", this.onSubmitStart)
    this.element.addEventListener("turbo:submit-end", this.onSubmitEnd)
  }

  disconnect() {
    this.element.removeEventListener("turbo:submit-start", this.onSubmitStart)
    this.element.removeEventListener("turbo:submit-end", this.onSubmitEnd)
  }

  onSubmitStart = () => {
    this.submitTarget.classList.add("btn--submitting")
  }

  onSubmitEnd = (event) => {
    this.submitTarget.classList.remove("btn--submitting")

    if (event.detail.success) {
      this.submitTarget.classList.add("btn--success")
      setTimeout(() => {
        this.submitTarget.classList.remove("btn--success")
      }, 1500)
    } else {
      this.submitTarget.classList.add("btn--error")
      setTimeout(() => {
        this.submitTarget.classList.remove("btn--error")
      }, 1500)
    }
  }
}
```

### CSS

```css
@layer components {
  .project-form {
    max-inline-size: 32rem;
  }

  .form-group {
    margin-block-end: var(--space-lg);
  }

  .form-label {
    display: block;
    font-weight: 600;
    margin-block-end: var(--space-xs);
    font-size: var(--text-sm);
  }

  .form-input {
    inline-size: 100%;
    padding: var(--space-sm) var(--space-md);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    font-size: var(--text-base);
    background: var(--color-surface);
    transition: border-color 0.15s, box-shadow 0.15s;

    &:focus {
      outline: none;
      border-color: var(--color-primary);
      box-shadow: 0 0 0 2px color-mix(in oklch, var(--color-primary), transparent 80%);
    }

    &:invalid:not(:placeholder-shown) {
      border-color: var(--color-danger);
    }
  }

  textarea.form-input {
    resize: vertical;
    min-block-size: 6rem;
  }

  .form-actions {
    display: flex;
    justify-content: flex-end;
    gap: var(--space-sm);
    padding-block-start: var(--space-md);
  }

  /* Button states */
  .btn {
    display: inline-flex;
    align-items: center;
    gap: var(--space-sm);
    padding: var(--space-sm) var(--space-lg);
    border: none;
    border-radius: var(--radius-md);
    font-weight: 600;
    font-size: var(--text-base);
    cursor: pointer;
    transition: background-color 0.15s, box-shadow 0.15s;
    position: relative;

    &.btn--primary {
      background: var(--color-primary);
      color: white;

      &:hover {
        background: color-mix(in oklch, var(--color-primary), white 15%);
      }

      &:active {
        background: color-mix(in oklch, var(--color-primary), black 10%);
      }
    }
  }

  /* Submitting state: spinner next to text */
  .btn--submitting {
    pointer-events: none;
    opacity: 0.8;

    &::after {
      content: "";
      inline-size: 1em;
      block-size: 1em;
      border: 2px solid currentColor;
      border-inline-end-color: transparent;
      border-radius: 50%;
      animation: spin 0.6s linear infinite;
    }
  }

  @keyframes spin {
    to { rotate: 360deg; }
  }

  /* Success state: brief green flash */
  .btn--success {
    background: var(--color-success) !important;
    animation: success-pulse 0.3s ease-out;
  }

  @keyframes success-pulse {
    0% { scale: 1; }
    50% { scale: 1.05; }
    100% { scale: 1; }
  }

  /* Error state: brief red flash */
  .btn--error {
    background: var(--color-danger) !important;
    animation: error-shake 0.3s ease-out;
  }

  @keyframes error-shake {
    0%, 100% { translate: 0; }
    20% { translate: -4px; }
    40% { translate: 4px; }
    60% { translate: -2px; }
    80% { translate: 2px; }
  }

  /* Disabled state from Turbo */
  .btn[disabled] {
    opacity: 0.6;
    cursor: not-allowed;
  }

  @media (prefers-reduced-motion: reduce) {
    .btn--submitting::after { animation: none; }
    .btn--success { animation: none; }
    .btn--error { animation: none; }
  }
}
```

---

## Example 5: Lazy-Loaded Dashboard Card with Skeleton Placeholder

A dashboard page with multiple lazy-loaded stat cards, each showing a skeleton placeholder until its data loads.

### Dashboard Page

```erb
<%# app/views/dashboards/show.html.erb %>
<h1>Dashboard</h1>

<div class="dashboard-grid">
  <%
    cards = [
      { id: "revenue",   title: "Revenue",        path: dashboard_revenue_path },
      { id: "users",     title: "Active Users",   path: dashboard_users_path },
      { id: "orders",    title: "Orders Today",   path: dashboard_orders_path },
      { id: "conversion", title: "Conversion Rate", path: dashboard_conversion_path }
    ]
  %>

  <% cards.each do |card| %>
    <%= turbo_frame_tag "dashboard_#{card[:id]}",
      src: card[:path],
      loading: :lazy,
      aria: { label: card[:title] } do %>

      <div class="stat-card-skeleton" aria-hidden="true">
        <div class="stat-card-skeleton-header">
          <div class="skeleton skeleton-text" style="inline-size: 50%;"></div>
        </div>
        <div class="stat-card-skeleton-value">
          <div class="skeleton skeleton-heading" style="inline-size: 40%;"></div>
        </div>
        <div class="stat-card-skeleton-chart">
          <div class="skeleton" style="inline-size: 100%; block-size: 40px;"></div>
        </div>
      </div>
      <span class="sr-only" role="status">Loading <%= card[:title].downcase %>...</span>
    <% end %>
  <% end %>
</div>
```

### Stat Card Partial (loaded into the frame)

```erb
<%# app/views/dashboards/_stat_card.html.erb %>
<div class="stat-card">
  <div class="stat-card-header">
    <h3 class="stat-card-title"><%= title %></h3>
    <span class="stat-card-trend stat-card-trend--<%= trend_direction %>">
      <%= trend_direction == "up" ? "+" : "" %><%= trend_percentage %>%
    </span>
  </div>
  <div class="stat-card-value"><%= value %></div>
  <div class="stat-card-chart">
    <%# Mini sparkline or chart %>
    <svg class="sparkline" viewBox="0 0 100 30" preserveAspectRatio="none">
      <polyline points="<%= sparkline_points %>" fill="none" stroke="currentColor" stroke-width="1.5"/>
    </svg>
  </div>
</div>
```

### CSS

```css
@layer components {
  .dashboard-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: var(--space-lg);
    margin-block-start: var(--space-xl);
  }

  /* Stat card (loaded content) */
  .stat-card {
    padding: var(--space-lg);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    background: var(--color-surface);

    /* Fade in when loaded */
    transition: opacity 0.2s ease-out;

    @starting-style {
      opacity: 0;
    }
  }

  .stat-card-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-block-end: var(--space-sm);
  }

  .stat-card-title {
    font-size: var(--text-sm);
    font-weight: 500;
    color: var(--color-text-muted);
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  .stat-card-trend {
    font-size: var(--text-sm);
    font-weight: 600;
    padding: var(--space-xs) var(--space-sm);
    border-radius: var(--radius-sm);

    &.stat-card-trend--up {
      color: oklch(40% 0.15 145);
      background: color-mix(in oklch, oklch(65% 0.2 145), transparent 90%);
    }

    &.stat-card-trend--down {
      color: oklch(40% 0.18 25);
      background: color-mix(in oklch, oklch(55% 0.22 25), transparent 90%);
    }
  }

  .stat-card-value {
    font-size: var(--text-2xl);
    font-weight: 700;
    margin-block-end: var(--space-md);
  }

  .stat-card-chart {
    color: var(--color-primary);
  }

  .sparkline {
    inline-size: 100%;
    block-size: 40px;
  }

  /* Skeleton placeholder (matches stat card dimensions) */
  .stat-card-skeleton {
    padding: var(--space-lg);
    border: 1px solid var(--color-border);
    border-radius: var(--radius-md);
    min-block-size: 160px;
  }

  .stat-card-skeleton-header {
    margin-block-end: var(--space-md);
  }

  .stat-card-skeleton-value {
    margin-block-end: var(--space-md);
  }

  .stat-card-skeleton-chart {
    margin-block-start: auto;
  }

  /* Skeleton base styles */
  .skeleton {
    background: linear-gradient(
      90deg,
      var(--color-border) 25%,
      color-mix(in oklch, var(--color-border), white 30%) 50%,
      var(--color-border) 75%
    );
    background-size: 200% 100%;
    border-radius: var(--radius-sm);
    animation: skeleton-shimmer 1.5s ease-in-out 0.2s infinite backwards;
  }

  .skeleton-text {
    block-size: 0.875rem;
  }

  .skeleton-heading {
    block-size: 1.75rem;
  }

  @keyframes skeleton-shimmer {
    0% { background-position: 200% 0; }
    100% { background-position: -200% 0; }
  }

  @media (prefers-reduced-motion: reduce) {
    .skeleton {
      animation: none;
      background: var(--color-border);
    }

    .stat-card {
      transition: none;
    }
  }
}
```

### Controller

```ruby
# app/controllers/dashboards_controller.rb
class DashboardsController < ApplicationController
  def show
    # The main page loads instantly with skeleton placeholders.
    # Each stat card loads lazily via its own endpoint.
  end

  def revenue
    @revenue = Current.account.revenue_stats
    render partial: "dashboards/stat_card", locals: {
      title: "Revenue",
      value: number_to_currency(@revenue.current),
      trend_direction: @revenue.trend_direction,
      trend_percentage: @revenue.trend_percentage,
      sparkline_points: @revenue.sparkline_points
    }
  end

  # Similar for users, orders, conversion...
end
```

The dashboard page loads instantly because the initial response contains only skeleton placeholders. Each stat card loads independently via a lazy Turbo Frame, meaning slow database queries for one card do not block the others. The skeletons match the final layout dimensions, preventing layout shift. When each card loads, it fades in with `@starting-style`, creating a polished staggered-load effect.
