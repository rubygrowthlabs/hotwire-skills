---
title: "Lifecycle: connect and disconnect"
skill: stimulus-controllers
topic: Stimulus controller lifecycle hooks
---

# Lifecycle: connect and disconnect

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [The Three Lifecycle Hooks](#the-three-lifecycle-hooks)
  - [Setup and Teardown Symmetry](#setup-and-teardown-symmetry)
  - [Re-entry Handling During Turbo Navigations](#re-entry-handling-during-turbo-navigations)
  - [MutationObserver Patterns for DOM-Driven Reactivity](#mutationobserver-patterns-for-dom-driven-reactivity)
  - [Third-Party Library Integration](#third-party-library-integration)
- [Pattern Card](#pattern-card)

## Overview

Stimulus controllers have three lifecycle hooks that fire at specific moments:

| Hook | When It Fires | Fires Multiple Times? |
|------|---------------|----------------------|
| `initialize()` | Once, when the controller is first instantiated | No |
| `connect()` | Each time the controller's element is inserted into the DOM | Yes |
| `disconnect()` | Each time the controller's element is removed from the DOM | Yes |

The most common mistake is treating `connect()` like a constructor. Because Turbo caches pages and restores them from cache, `connect()` fires every time the user navigates back to a cached page. Controllers must be written to handle multiple connect/disconnect cycles cleanly.

**When to use each hook:**

- `initialize()`: One-time setup that should never repeat. Binding methods, creating non-DOM state. Rarely needed since most setup depends on the DOM being present.
- `connect()`: DOM-dependent setup. Adding event listeners, starting observers, initializing third-party libraries that need a DOM element.
- `disconnect()`: Tearing down everything `connect()` created. Removing event listeners, stopping observers, destroying third-party library instances.

## Implementation

### The Three Lifecycle Hooks

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  // Fires once per controller instance, before the first connect().
  // The element is available but may not be in the DOM yet.
  initialize() {
    // Bind methods that will be used as event listeners.
    // This avoids creating new function references on each connect().
    this.handleScroll = this.handleScroll.bind(this)
    this.handleResize = this.handleResize.bind(this)
  }

  // Fires each time the element appears in the DOM.
  // Safe to access targets, values, and outlets here.
  connect() {
    window.addEventListener("scroll", this.handleScroll)
    window.addEventListener("resize", this.handleResize)
  }

  // Fires each time the element is removed from the DOM.
  // Must undo everything connect() did.
  disconnect() {
    window.removeEventListener("scroll", this.handleScroll)
    window.removeEventListener("resize", this.handleResize)
  }

  handleScroll() {
    // ...
  }

  handleResize() {
    // ...
  }
}
```

### Setup and Teardown Symmetry

Every resource acquired in `connect()` must be released in `disconnect()`. This table shows common patterns:

| Resource | Setup in connect() | Cleanup in disconnect() |
|----------|-------------------|------------------------|
| Event listener | `addEventListener()` | `removeEventListener()` |
| Timer | `setInterval()` / `setTimeout()` | `clearInterval()` / `clearTimeout()` |
| IntersectionObserver | `new IntersectionObserver(); .observe()` | `.disconnect()` |
| MutationObserver | `new MutationObserver(); .observe()` | `.disconnect()` |
| ResizeObserver | `new ResizeObserver(); .observe()` | `.disconnect()` |
| AbortController | `new AbortController()` | `.abort()` |
| Third-party instance | `new Library(this.element)` | `.destroy()` |

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    // Timer
    this.refreshInterval = setInterval(() => this.refresh(), 30000)

    // AbortController for fetch requests
    this.abortController = new AbortController()

    // IntersectionObserver
    this.observer = new IntersectionObserver(
      (entries) => this.handleIntersection(entries),
      { threshold: 0.1 }
    )
    this.observer.observe(this.element)
  }

  disconnect() {
    // Timer
    clearInterval(this.refreshInterval)

    // AbortController
    this.abortController.abort()

    // IntersectionObserver
    this.observer.disconnect()
  }

  async refresh() {
    try {
      const response = await fetch(this.element.dataset.url, {
        signal: this.abortController.signal
      })
      // handle response
    } catch (error) {
      if (error.name !== "AbortError") throw error
    }
  }

  handleIntersection(entries) {
    // ...
  }
}
```

### Re-entry Handling During Turbo Navigations

When a user navigates away and then presses the back button, Turbo restores the page from cache. This means `connect()` fires again on a controller that previously had `disconnect()` called. Controllers must handle this gracefully.

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { loaded: Boolean }

  connect() {
    // Guard against re-initialization of expensive operations
    if (!this.loadedValue) {
      this.performExpensiveSetup()
      this.loadedValue = true
    }

    // Always re-attach event listeners (they were removed in disconnect)
    this.handleClick = this.handleClick.bind(this)
    this.element.addEventListener("click", this.handleClick)
  }

  disconnect() {
    this.element.removeEventListener("click", this.handleClick)
    // Note: we do NOT reset loadedValue here, so the expensive
    // setup is skipped on re-entry from Turbo cache.
  }

  performExpensiveSetup() {
    // One-time setup that should not repeat on back-navigation
  }

  handleClick() {
    // ...
  }
}
```

### MutationObserver Patterns for DOM-Driven Reactivity

Use MutationObserver when you need to react to DOM changes outside your controller's direct control (e.g., Turbo Stream updates adding content).

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["list"]

  connect() {
    this.mutationObserver = new MutationObserver(
      (mutations) => this.handleMutations(mutations)
    )
    this.mutationObserver.observe(this.listTarget, {
      childList: true,
      subtree: false
    })
  }

  disconnect() {
    this.mutationObserver.disconnect()
  }

  handleMutations(mutations) {
    for (const mutation of mutations) {
      for (const node of mutation.addedNodes) {
        if (node.nodeType === Node.ELEMENT_NODE) {
          this.animateIn(node)
        }
      }
    }
  }

  animateIn(element) {
    element.animate(
      [{ opacity: 0, transform: "translateY(-10px)" }, { opacity: 1, transform: "translateY(0)" }],
      { duration: 200, easing: "ease-out" }
    )
  }
}
```

**Prefer target callbacks over MutationObserver when possible.** Target callbacks (`itemTargetConnected`) are cleaner and more idiomatic. Use MutationObserver only when the changing elements are not Stimulus targets.

### Third-Party Library Integration

When wrapping a third-party library, create the instance in `connect()` and destroy it in `disconnect()`:

```javascript
import { Controller } from "@hotwired/stimulus"
import flatpickr from "flatpickr"

export default class extends Controller {
  static values = {
    enableTime: { type: Boolean, default: false },
    dateFormat: { type: String, default: "Y-m-d" }
  }

  connect() {
    this.picker = flatpickr(this.element, {
      enableTime: this.enableTimeValue,
      dateFormat: this.dateFormatValue
    })
  }

  disconnect() {
    this.picker.destroy()
  }
}
```

```html
<input type="text"
       data-controller="datepicker"
       data-datepicker-enable-time-value="true"
       data-datepicker-date-format-value="Y-m-d H:i">
```

## Pattern Card

### GOOD: Symmetric setup and teardown

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  initialize() {
    this.handleResize = this.handleResize.bind(this)
  }

  connect() {
    this.observer = new ResizeObserver(() => this.handleResize())
    this.observer.observe(this.element)

    this.interval = setInterval(() => this.poll(), 5000)
  }

  disconnect() {
    this.observer.disconnect()
    clearInterval(this.interval)
  }

  handleResize() { /* ... */ }
  poll() { /* ... */ }
}
```

Every resource created in `connect()` is destroyed in `disconnect()`. The controller survives multiple connect/disconnect cycles without leaks.

### BAD: Setup without cleanup causing leaks

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  connect() {
    // Leak: new ResizeObserver on every connect, never disconnected
    new ResizeObserver(() => this.handleResize()).observe(this.element)

    // Leak: interval never cleared
    setInterval(() => this.poll(), 5000)

    // Leak: event listener added but never removed
    window.addEventListener("resize", () => this.handleResize())
  }

  handleResize() { /* ... */ }
  poll() { /* ... */ }
}
```

Each Turbo navigation creates another observer, another interval, and another event listener. After 10 navigations, there are 10 intervals polling simultaneously and 10 resize listeners firing on every resize.
