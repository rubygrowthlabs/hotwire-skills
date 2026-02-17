---
title: "Targets and Target Callbacks"
skill: stimulus-controllers
topic: Stimulus Targets API with targetConnected/targetDisconnected callbacks
---

# Targets and Target Callbacks

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Declaring and Using Targets](#declaring-and-using-targets)
  - [Target Properties](#target-properties)
  - [Target Callbacks for Dynamic Content](#target-callbacks-for-dynamic-content)
  - [Targets With Turbo Streams](#targets-with-turbo-streams)
  - [Nested Controllers and Target Scope](#nested-controllers-and-target-scope)
- [Pattern Card](#pattern-card)

## Overview

Targets are Stimulus's way of referencing DOM elements within a controller. They replace `querySelector` with a declarative, scoped system.

For every target name, Stimulus generates three properties:

| Property | Type | Description |
|----------|------|-------------|
| `this.nameTarget` | `Element` | The first matching element (throws if missing) |
| `this.nameTargets` | `Element[]` | All matching elements |
| `this.hasNameTarget` | `Boolean` | Whether at least one matching element exists |

Stimulus also provides two callbacks for dynamically added or removed targets:

| Callback | When It Fires |
|----------|---------------|
| `nameTargetConnected(element)` | When a matching target element is added to the DOM |
| `nameTargetDisconnected(element)` | When a matching target element is removed from the DOM |

These callbacks are essential for controllers that manage lists or respond to Turbo Stream updates.

## Implementation

### Declaring and Using Targets

Targets are declared as a static array of names and connected via `data-*-target` attributes in the HTML:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "output", "error"]

  validate() {
    const value = this.inputTarget.value

    if (value.length < 3) {
      this.errorTarget.textContent = "Too short"
      this.errorTarget.hidden = false
    } else {
      this.errorTarget.hidden = true
      this.outputTarget.textContent = value
    }
  }
}
```

```html
<div data-controller="validator">
  <input data-validator-target="input"
         data-action="input->validator#validate"
         type="text">

  <p data-validator-target="error" hidden></p>
  <p data-validator-target="output"></p>
</div>
```

### Target Properties

Use the generated properties to interact with targets:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["item", "counter", "emptyMessage"]

  connect() {
    this.updateCount()
  }

  updateCount() {
    // this.itemTargets returns all matching elements as an array
    const count = this.itemTargets.length

    // this.counterTarget returns the first (and only, in this case) match
    this.counterTarget.textContent = count

    // this.hasEmptyMessageTarget checks existence before access
    if (this.hasEmptyMessageTarget) {
      this.emptyMessageTarget.hidden = count > 0
    }
  }

  remove(event) {
    event.target.closest("[data-list-target='item']").remove()
    this.updateCount()
  }
}
```

**Important:** `this.nameTarget` throws an error if no matching element exists. Always use `this.hasNameTarget` to guard optional targets:

```javascript
// SAFE: Guard before access
if (this.hasErrorTarget) {
  this.errorTarget.textContent = message
}

// UNSAFE: Throws if no error target exists
this.errorTarget.textContent = message
```

### Target Callbacks for Dynamic Content

Target callbacks fire when target elements are added to or removed from the DOM. This is ideal for responding to Turbo Stream updates, lazy-loaded content, or user actions that add/remove elements.

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["notification"]
  static values = { count: { type: Number, default: 0 } }

  // Fires when a notification target is added to the DOM
  notificationTargetConnected(element) {
    // Idempotent: always recount from the DOM
    this.countValue = this.notificationTargets.length
    this.animateIn(element)
  }

  // Fires when a notification target is removed from the DOM
  notificationTargetDisconnected(element) {
    this.countValue = this.notificationTargets.length
  }

  countValueChanged() {
    this.element.querySelector("[data-badge]").textContent = this.countValue
  }

  animateIn(element) {
    element.animate(
      [{ opacity: 0 }, { opacity: 1 }],
      { duration: 300, easing: "ease-out" }
    )
  }

  dismiss(event) {
    const notification = event.target.closest("[data-notifications-target='notification']")
    notification.remove()
  }
}
```

```html
<div data-controller="notifications" data-notifications-count-value="2">
  <span data-badge>2</span> notifications

  <div data-notifications-target="notification">
    <p>You have a new message</p>
    <button data-action="notifications#dismiss">Dismiss</button>
  </div>

  <div data-notifications-target="notification">
    <p>Your export is ready</p>
    <button data-action="notifications#dismiss">Dismiss</button>
  </div>

  <!-- Turbo Stream can append new notifications here, and
       notificationTargetConnected will fire automatically -->
</div>
```

**Key rules for target callbacks:**

1. Callbacks must be **idempotent**. They may fire multiple times (e.g., when Turbo morphs the page).
2. Use `this.nameTargets.length` to recount rather than incrementing/decrementing a counter.
3. The element is fully connected when `targetConnected` fires -- targets, values, and actions on the element are available.
4. The element is still in the DOM when `targetDisconnected` fires, but about to be removed.

### Targets With Turbo Streams

Target callbacks pair perfectly with Turbo Streams. When a stream appends, prepends, or replaces an element, the target callbacks fire automatically:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["message"]
  static values = { autoScroll: { type: Boolean, default: true } }

  messageTargetConnected(element) {
    if (this.autoScrollValue) {
      element.scrollIntoView({ behavior: "smooth", block: "end" })
    }
  }
}
```

```erb
<%# app/views/messages/_message.html.erb %>
<div data-chat-target="message" id="<%= dom_id(message) %>">
  <strong><%= message.user.name %></strong>
  <p><%= message.body %></p>
</div>

<%# Turbo Stream broadcast %>
<%# When a new message is appended, messageTargetConnected fires automatically %>
<%= turbo_stream.append "messages", partial: "messages/message", locals: { message: message } %>
```

### Nested Controllers and Target Scope

Targets are scoped to their nearest controller element. A target in a nested controller does not appear in the parent controller's targets:

```html
<div data-controller="parent">
  <!-- This item target belongs to parent -->
  <div data-parent-target="item">
    <div data-controller="child">
      <!-- This item target belongs to child, NOT parent -->
      <div data-child-target="item">...</div>
    </div>
  </div>
</div>
```

```javascript
// In the parent controller:
this.itemTargets // => [outer div only]

// In the child controller:
this.itemTargets // => [inner div only]
```

If you need the parent to access elements inside a nested controller, use **outlets** instead of targets.

## Pattern Card

### GOOD: Target callback for dynamic list

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["item", "count", "empty"]

  itemTargetConnected() {
    this.updateUI()
  }

  itemTargetDisconnected() {
    this.updateUI()
  }

  updateUI() {
    const count = this.itemTargets.length
    this.countTarget.textContent = count
    this.emptyTarget.hidden = count > 0
  }
}
```

The controller automatically reacts to items being added or removed by Turbo Streams, DOM manipulation, or user actions. The `updateUI` method is idempotent and safe to call from either callback.

### BAD: Manual querySelector after mutation

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  connect() {
    this.mutationObserver = new MutationObserver(() => {
      // Fragile: querying by CSS class instead of targets
      const items = this.element.querySelectorAll(".list-item")
      this.element.querySelector(".count").textContent = items.length
      this.element.querySelector(".empty").hidden = items.length > 0
    })
    this.mutationObserver.observe(this.element, { childList: true, subtree: true })
  }

  disconnect() {
    this.mutationObserver.disconnect()
  }
}
```

MutationObserver is overkill here and fires for every DOM change (attribute changes, text changes), not just target additions. The querySelector calls are fragile and coupled to CSS class names rather than semantic targets.
