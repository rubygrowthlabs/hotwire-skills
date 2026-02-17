---
title: "Frame Loading Spinners"
categories:
  - ux
  - turbo
  - loading-state
tags:
  - turbo-frames
  - skeleton-screen
  - spinner
  - aria-busy
  - lazy-loading
  - layout-shift
description: >-
  Loading feedback specific to Turbo Frame loading. Covers CSS-only spinners
  inside lazy-loaded frame placeholders, turbo:before-fetch-request/response events,
  skeleton screens with CSS animations, aria-busy for accessibility, and
  fixed-height placeholders to prevent layout shift.
---

# Frame Loading Spinners

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [CSS-Only Spinner Inside Lazy-Loaded Frames](#css-only-spinner-inside-lazy-loaded-frames)
  - [Frame Loading Events for Custom Indicators](#frame-loading-events-for-custom-indicators)
  - [Skeleton Screens with CSS Animations](#skeleton-screens-with-css-animations)
  - [aria-busy for Accessibility During Loading](#aria-busy-for-accessibility-during-loading)
  - [Preventing Layout Shift with Fixed-Height Placeholders](#preventing-layout-shift-with-fixed-height-placeholders)
  - [Combining Skeletons with Lazy Frames](#combining-skeletons-with-lazy-frames)
- [Pattern Card](#pattern-card)

## Overview

Turbo Frames fetch content asynchronously, creating a gap between the frame appearing on the page and its content loading. During this gap, users need feedback that something is happening. The three approaches, in order of preference:

1. **Skeleton screens**: Gray placeholder shapes that mimic the eventual content layout. Best for content-heavy frames where the layout is predictable (cards, lists, profiles).
2. **Inline spinners**: A small animated indicator inside the frame. Best for compact frames where the content layout is unpredictable (search results, dynamic widgets).
3. **Overlay indicators**: A loading overlay on top of existing content. Best for frames that reload (pagination, filtering) where the old content should remain visible but dimmed.

All three approaches should be CSS-only. The placeholder content goes inside the `turbo_frame_tag` and is replaced when the frame loads. Turbo automatically sets `aria-busy="true"` on frames while loading, which you should use for both styling and accessibility.

## Implementation

### CSS-Only Spinner Inside Lazy-Loaded Frames

Place spinner markup inside the frame tag. Turbo replaces it when the content loads.

```erb
<%# app/views/dashboards/show.html.erb %>
<%= turbo_frame_tag "recent_activity", src: activity_path, loading: :lazy do %>
  <div class="frame-spinner">
    <div class="spinner" role="status">
      <span class="sr-only">Loading activity...</span>
    </div>
  </div>
<% end %>
```

```css
@layer components {
  .frame-spinner {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: var(--space-xl);
    min-block-size: 120px;
  }

  .spinner {
    inline-size: 1.5rem;
    block-size: 1.5rem;
    border: 2px solid var(--color-border);
    border-block-start-color: var(--color-primary);
    border-radius: 50%;
    animation: spin 0.6s linear infinite;

    /* Delay showing to avoid flash on fast loads */
    opacity: 0;
    animation:
      fadeIn 0.1s ease-in 0.2s forwards,
      spin 0.6s linear 0.2s infinite;
  }

  @keyframes spin {
    to { rotate: 360deg; }
  }

  @keyframes fadeIn {
    to { opacity: 1; }
  }

  /* Smaller spinner variant for inline use */
  .spinner--sm {
    inline-size: 1rem;
    block-size: 1rem;
    border-width: 1.5px;
  }

  /* Spinner that matches text color for inline use */
  .spinner--inline {
    inline-size: 1em;
    block-size: 1em;
    border-color: currentColor;
    border-block-start-color: transparent;
    vertical-align: middle;
    margin-inline-start: var(--space-xs);
  }

  @media (prefers-reduced-motion: reduce) {
    .spinner {
      animation: fadeIn 0.1s ease-in 0.2s forwards;
      border-block-start-color: var(--color-primary);
      /* Show as a static colored circle instead of spinning */
    }
  }
}
```

### Frame Loading Events for Custom Indicators

Turbo fires events on frames during the fetch lifecycle. Use these for custom loading behavior that CSS alone cannot handle.

```javascript
// app/javascript/controllers/frame_loading_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["overlay"]

  connect() {
    this.element.addEventListener("turbo:before-fetch-request", this.showLoading)
    this.element.addEventListener("turbo:before-fetch-response", this.hideLoading)
    this.element.addEventListener("turbo:fetch-request-error", this.showError)
  }

  disconnect() {
    this.element.removeEventListener("turbo:before-fetch-request", this.showLoading)
    this.element.removeEventListener("turbo:before-fetch-response", this.hideLoading)
    this.element.removeEventListener("turbo:fetch-request-error", this.showError)
  }

  showLoading = () => {
    this.overlayTarget.hidden = false
  }

  hideLoading = () => {
    this.overlayTarget.hidden = true
  }

  showError = () => {
    this.overlayTarget.hidden = true
    // Optionally show an error state
    this.element.classList.add("frame-error")
  }
}
```

```erb
<%# Frame with reload overlay (for pagination, filtering) %>
<%= turbo_frame_tag "search_results",
  data: { controller: "frame-loading" } do %>
  <div class="frame-overlay" data-frame-loading-target="overlay" hidden>
    <div class="spinner"></div>
  </div>
  <%= render "results", results: @results %>
<% end %>
```

```css
@layer components {
  turbo-frame {
    position: relative;
    display: block;
  }

  .frame-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    background: color-mix(in oklch, var(--color-surface), transparent 30%);
    border-radius: inherit;
    z-index: 1;

    /* Delayed appearance */
    opacity: 0;
    animation: fadeIn 0.15s ease-in 0.2s forwards;
  }

  .frame-overlay[hidden] {
    display: none;
  }
}
```

However, in most cases you can use the `aria-busy` attribute that Turbo sets automatically, without any custom JavaScript:

```css
@layer components {
  /* Turbo automatically sets aria-busy during frame loading */
  turbo-frame[aria-busy="true"] {
    position: relative;
    pointer-events: none;

    /* Dim the existing content */
    & > *:not(.frame-placeholder) {
      opacity: 0.5;
      transition: opacity 0.15s;
    }
  }

  /* Show spinner when frame is busy */
  turbo-frame[aria-busy="true"]::after {
    content: "";
    position: absolute;
    inset-block-start: 50%;
    inset-inline-start: 50%;
    translate: -50% -50%;
    inline-size: 1.5rem;
    block-size: 1.5rem;
    border: 2px solid var(--color-border);
    border-block-start-color: var(--color-primary);
    border-radius: 50%;

    /* Delay + spin */
    opacity: 0;
    animation:
      fadeIn 0.1s ease-in 0.2s forwards,
      spin 0.6s linear 0.2s infinite;
  }
}
```

### Skeleton Screens with CSS Animations

Skeleton screens are gray placeholder shapes that mimic the layout of the content that will load. They are more informative than spinners because they set expectations about what the content will look like.

```css
@layer components {
  .skeleton {
    background: linear-gradient(
      90deg,
      var(--color-border) 25%,
      color-mix(in oklch, var(--color-border), white 30%) 50%,
      var(--color-border) 75%
    );
    background-size: 200% 100%;
    animation: skeleton-shimmer 1.5s ease-in-out infinite;
    border-radius: var(--radius-sm);

    /* Delay before shimmer starts */
    animation-delay: 0.2s;
    animation-fill-mode: backwards;
  }

  @keyframes skeleton-shimmer {
    0% { background-position: 200% 0; }
    100% { background-position: -200% 0; }
  }

  /* Pre-defined skeleton shapes */
  .skeleton-text {
    block-size: 1em;
    inline-size: 100%;
    margin-block-end: var(--space-sm);

    &:last-child {
      inline-size: 60%;
    }
  }

  .skeleton-heading {
    block-size: 1.5em;
    inline-size: 40%;
    margin-block-end: var(--space-md);
  }

  .skeleton-avatar {
    inline-size: 2.5rem;
    block-size: 2.5rem;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .skeleton-thumbnail {
    inline-size: 100%;
    aspect-ratio: 16 / 9;
    border-radius: var(--radius-md);
  }

  .skeleton-button {
    inline-size: 6rem;
    block-size: 2.25rem;
    border-radius: var(--radius-md);
  }

  @media (prefers-reduced-motion: reduce) {
    .skeleton {
      animation: none;
      background: var(--color-border);
    }
  }
}
```

```erb
<%# Skeleton placeholder for a user card %>
<%= turbo_frame_tag "user_profile", src: user_profile_path, loading: :lazy do %>
  <div class="user-card-skeleton" aria-hidden="true">
    <div class="flex items-center gap-md">
      <div class="skeleton skeleton-avatar"></div>
      <div style="flex: 1;">
        <div class="skeleton skeleton-text" style="inline-size: 50%;"></div>
        <div class="skeleton skeleton-text" style="inline-size: 30%;"></div>
      </div>
    </div>
    <div style="margin-block-start: var(--space-md);">
      <div class="skeleton skeleton-text"></div>
      <div class="skeleton skeleton-text"></div>
      <div class="skeleton skeleton-text" style="inline-size: 75%;"></div>
    </div>
  </div>
<% end %>
```

```erb
<%# Skeleton placeholder for a stats card %>
<%= turbo_frame_tag "dashboard_stats", src: stats_path, loading: :lazy do %>
  <div class="stats-skeleton" aria-hidden="true">
    <div class="grid gap-md" style="grid-template-columns: repeat(3, 1fr);">
      <% 3.times do %>
        <div class="p-md border rounded-md">
          <div class="skeleton skeleton-text" style="inline-size: 40%;"></div>
          <div class="skeleton skeleton-heading" style="inline-size: 60%;"></div>
        </div>
      <% end %>
    </div>
  </div>
<% end %>
```

### aria-busy for Accessibility During Loading

Turbo automatically sets `aria-busy="true"` on frames while they are loading. This attribute serves two purposes: it tells assistive technology that the region is updating, and it provides a CSS hook for styling.

```css
@layer components {
  /* Visual indicator that the frame is loading */
  turbo-frame[aria-busy="true"] {
    min-block-size: 4rem; /* Prevent collapse */
  }

  /* Announce loading state to screen readers */
  turbo-frame[aria-busy="true"]::before {
    content: "Loading content";
    position: absolute;
    inline-size: 1px;
    block-size: 1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
  }
}
```

For frames that update frequently (filtering, pagination), add `aria-live` to announce changes:

```erb
<%= turbo_frame_tag "search_results", aria: { live: "polite", atomic: "false" } do %>
  <%= render "results", results: @results %>
<% end %>
```

When combining with skeletons, use `aria-hidden="true"` on the skeleton so screen readers do not try to read the placeholder shapes:

```erb
<%= turbo_frame_tag "content", src: content_path, loading: :lazy do %>
  <div aria-hidden="true" class="skeleton-container">
    <%# Skeleton shapes here %>
  </div>
  <span class="sr-only" role="status">Loading content...</span>
<% end %>
```

### Preventing Layout Shift with Fixed-Height Placeholders

When a frame loads, the placeholder content is replaced by the actual content. If the actual content is a different height, the page layout shifts, which is jarring and hurts Core Web Vitals (CLS metric).

```css
@layer components {
  /* Fixed-height frame placeholder prevents layout shift */
  .frame-placeholder {
    /* Match the expected height of the loaded content */
    min-block-size: var(--frame-min-height, 200px);
    display: flex;
    align-items: center;
    justify-content: center;
  }

  /* Common frame heights for different content types */
  .frame-placeholder--card {
    --frame-min-height: 180px;
  }

  .frame-placeholder--list {
    --frame-min-height: 320px;
  }

  .frame-placeholder--table {
    --frame-min-height: 400px;
  }

  .frame-placeholder--compact {
    --frame-min-height: 80px;
  }
}
```

```erb
<%# Set a min-height that matches the expected content %>
<%= turbo_frame_tag "activity_feed", src: activity_path, loading: :lazy do %>
  <div class="frame-placeholder frame-placeholder--list">
    <div class="loading-indicator">
      <div class="loading-dots">
        <span></span><span></span><span></span>
      </div>
      <span class="sr-only">Loading activity feed...</span>
    </div>
  </div>
<% end %>
```

For content with highly variable height, use `content-visibility: auto` to let the browser skip rendering off-screen content:

```css
@layer components {
  /* For long lists inside frames, skip rendering off-screen items */
  turbo-frame .list-item {
    content-visibility: auto;
    contain-intrinsic-size: auto 60px; /* Expected height of each item */
  }
}
```

### Combining Skeletons with Lazy Frames

A complete example combining all techniques: lazy-loaded frame, skeleton placeholder, accessibility attributes, and layout shift prevention.

```erb
<%# app/views/dashboards/show.html.erb %>
<section class="dashboard-section">
  <h2>Recent Projects</h2>

  <%= turbo_frame_tag "recent_projects",
    src: recent_projects_path,
    loading: :lazy,
    aria: { label: "Recent projects" } do %>

    <div class="projects-skeleton" aria-hidden="true">
      <div class="grid gap-md" style="grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));">
        <% 4.times do %>
          <div class="project-card-skeleton border rounded-md p-md">
            <div class="skeleton skeleton-heading"></div>
            <div class="skeleton skeleton-text"></div>
            <div class="skeleton skeleton-text"></div>
            <div class="skeleton skeleton-text" style="inline-size: 40%;"></div>
            <div class="flex justify-between items-center" style="margin-block-start: var(--space-md);">
              <div class="skeleton skeleton-avatar" style="inline-size: 1.5rem; block-size: 1.5rem;"></div>
              <div class="skeleton skeleton-text" style="inline-size: 30%;"></div>
            </div>
          </div>
        <% end %>
      </div>
    </div>

    <span class="sr-only" role="status">Loading recent projects...</span>
  <% end %>
</section>
```

```css
@layer components {
  .projects-skeleton {
    min-block-size: 200px;
  }

  .project-card-skeleton {
    min-block-size: 160px;
  }
}
```

## Pattern Card

### GOOD: Skeleton Screen with aria-busy and Layout Shift Prevention

```erb
<%= turbo_frame_tag "user_profile",
  src: user_profile_path(@user),
  loading: :lazy,
  aria: { label: "User profile" } do %>

  <div class="frame-placeholder frame-placeholder--card" aria-hidden="true">
    <div class="flex items-center gap-md">
      <div class="skeleton skeleton-avatar"></div>
      <div style="flex: 1;">
        <div class="skeleton skeleton-text" style="inline-size: 40%;"></div>
        <div class="skeleton skeleton-text" style="inline-size: 25%;"></div>
      </div>
    </div>
  </div>
  <span class="sr-only" role="status">Loading user profile...</span>
<% end %>
```

```css
@layer components {
  .frame-placeholder--card { min-block-size: 180px; }

  .skeleton {
    background: linear-gradient(
      90deg,
      var(--color-border) 25%,
      color-mix(in oklch, var(--color-border), white 30%) 50%,
      var(--color-border) 75%
    );
    background-size: 200% 100%;
    animation: skeleton-shimmer 1.5s ease-in-out 0.2s infinite backwards;
    border-radius: var(--radius-sm);
  }

  @media (prefers-reduced-motion: reduce) {
    .skeleton { animation: none; background: var(--color-border); }
  }
}
```

The skeleton placeholder mimics the profile card layout, so users know what to expect. The `min-block-size` prevents layout shift when real content replaces the skeleton. `aria-hidden` prevents screen readers from reading placeholder shapes, while the `sr-only` status message announces loading. The shimmer animation is delayed 200ms and respects reduced motion. When the frame loads, Turbo replaces the placeholder seamlessly.

### BAD: Empty Frame with No Feedback

```erb
<%= turbo_frame_tag "user_profile",
  src: user_profile_path(@user),
  loading: :lazy %>
```

An empty lazy frame renders as nothing -- literally zero pixels. The user sees a gap in the page that suddenly fills with content when the request completes. There is no indication that content is coming, no accessible loading announcement, no layout reservation. The sudden appearance of content causes layout shift (poor CLS score), confuses screen reader users who receive no loading notification, and makes the page feel broken until the content appears. Even adding a simple text placeholder ("Loading...") would be better than nothing.
