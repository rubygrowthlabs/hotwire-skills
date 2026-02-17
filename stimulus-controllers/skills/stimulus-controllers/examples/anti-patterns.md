---
title: "Common Stimulus Anti-Patterns"
skill: stimulus-controllers
topic: 10 common Stimulus anti-patterns with corrected examples
---

# Common Stimulus Anti-Patterns

This document covers 10 common anti-patterns seen in Stimulus controllers, each with a problem description, a bad example, and a corrected example.

## Table of Contents

1. [Fat Controllers](#1-fat-controllers)
2. [Direct DOM Manipulation Instead of Values](#2-direct-dom-manipulation-instead-of-values)
3. [Missing disconnect Cleanup](#3-missing-disconnect-cleanup)
4. [Using querySelector Instead of Targets](#4-using-queryselector-instead-of-targets)
5. [Hardcoded Selectors](#5-hardcoded-selectors)
6. [Business Logic in Controllers](#6-business-logic-in-controllers)
7. [Tightly Coupled Controllers](#7-tightly-coupled-controllers)
8. [Ignoring Action Parameters](#8-ignoring-action-parameters)
9. [Not Using CSS Classes API](#9-not-using-css-classes-api)
10. [Synchronous Heavy Operations in connect](#10-synchronous-heavy-operations-in-connect)

---

## 1. Fat Controllers

**Problem:** A single controller handles multiple unrelated responsibilities, making it hard to test, reuse, or maintain. Stimulus controllers should be small and focused, ideally under 50 lines.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS - 150+ lines handling tabs, search, notifications, and tooltips
export default class extends Controller {
  static targets = [
    "tab", "tabPanel", "searchInput", "searchResults",
    "notification", "notificationBadge", "tooltip"
  ]

  selectTab(event) { /* 20 lines of tab logic */ }
  search(event) { /* 30 lines of search logic */ }
  dismissNotification(event) { /* 15 lines of notification logic */ }
  showTooltip(event) { /* 15 lines of tooltip logic */ }
  hideTooltip(event) { /* 10 lines of tooltip logic */ }
  positionTooltip(element) { /* 20 lines of positioning logic */ }
  filterResults(query) { /* 20 lines of filtering logic */ }
  updateBadge() { /* 10 lines of badge logic */ }
  // ... continues for 150+ lines
}
```

**Corrected:**

```javascript
// tabs_controller.js - 20 lines
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["tab", "panel"]
  static values = { index: { type: Number, default: 0 } }

  indexValueChanged() {
    this.showTab(this.indexValue)
  }

  select(event) {
    this.indexValue = this.tabTargets.indexOf(event.currentTarget)
  }

  showTab(index) {
    this.tabTargets.forEach((tab, i) => tab.classList.toggle("active", i === index))
    this.panelTargets.forEach((panel, i) => panel.hidden = i !== index)
  }
}

// search_controller.js - 15 lines
// notifications_controller.js - 20 lines
// tooltip_controller.js - 25 lines
```

Split into four focused controllers that compose through HTML structure or outlets.

---

## 2. Direct DOM Manipulation Instead of Values

**Problem:** Using instance variables and manually updating the DOM in every method, instead of using the Values API with reactive `valueChanged` callbacks.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  connect() {
    this.count = 0
    this.open = false
  }

  increment() {
    this.count++
    this.element.querySelector(".count").textContent = this.count
    this.element.querySelector(".badge").textContent = this.count
    this.element.querySelector(".empty").hidden = this.count > 0
  }

  toggle() {
    this.open = !this.open
    this.element.querySelector(".menu").hidden = !this.open
    this.element.querySelector(".icon").textContent = this.open ? "▼" : "▶"
  }
}
```

**Corrected:**

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["count", "badge", "empty", "menu", "icon"]
  static values = {
    count: { type: Number, default: 0 },
    open: { type: Boolean, default: false }
  }

  countValueChanged() {
    this.countTarget.textContent = this.countValue
    this.badgeTarget.textContent = this.countValue
    this.emptyTarget.hidden = this.countValue > 0
  }

  openValueChanged() {
    this.menuTarget.hidden = !this.openValue
    this.iconTarget.textContent = this.openValue ? "▼" : "▶"
  }

  increment() {
    this.countValue++
  }

  toggle() {
    this.openValue = !this.openValue
  }
}
```

State changes in one place (`increment`, `toggle`), UI updates react in one place (`valueChanged`). State survives Turbo cache restores.

---

## 3. Missing disconnect Cleanup

**Problem:** Event listeners, timers, or observers created in `connect()` are never cleaned up in `disconnect()`. Each Turbo navigation leaks another listener.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  connect() {
    // Leak: new anonymous listener on every connect, can never be removed
    window.addEventListener("scroll", () => this.handleScroll())

    // Leak: interval never cleared
    setInterval(() => this.refresh(), 5000)

    // Leak: observer never disconnected
    new IntersectionObserver(entries => this.observe(entries))
      .observe(this.element)
  }

  handleScroll() { /* ... */ }
  refresh() { /* ... */ }
  observe(entries) { /* ... */ }
}
```

**Corrected:**

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  initialize() {
    this.handleScroll = this.handleScroll.bind(this)
  }

  connect() {
    window.addEventListener("scroll", this.handleScroll)

    this.refreshInterval = setInterval(() => this.refresh(), 5000)

    this.observer = new IntersectionObserver(entries => this.observe(entries))
    this.observer.observe(this.element)
  }

  disconnect() {
    window.removeEventListener("scroll", this.handleScroll)
    clearInterval(this.refreshInterval)
    this.observer.disconnect()
  }

  handleScroll() { /* ... */ }
  refresh() { /* ... */ }
  observe(entries) { /* ... */ }
}
```

Every resource acquired in `connect()` has a corresponding cleanup in `disconnect()`. Method binding in `initialize()` ensures the same function reference is used for add and remove.

---

## 4. Using querySelector Instead of Targets

**Problem:** Querying elements by CSS selectors instead of using Stimulus targets. This is fragile, unscoped, and breaks when CSS classes change.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  submit() {
    const input = this.element.querySelector(".form-input")
    const error = this.element.querySelector(".error-message")
    const button = document.querySelector("#submit-btn")  // global query!

    if (input.value.length < 3) {
      error.textContent = "Too short"
      error.style.display = "block"
    } else {
      button.disabled = true
      // ...
    }
  }
}
```

**Corrected:**

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["input", "error", "submit"]

  submit() {
    if (this.inputTarget.value.length < 3) {
      this.errorTarget.textContent = "Too short"
      this.errorTarget.hidden = false
    } else {
      this.submitTarget.disabled = true
      // ...
    }
  }
}
```

```html
<div data-controller="form">
  <input data-form-target="input" type="text">
  <p data-form-target="error" hidden></p>
  <button data-form-target="submit" data-action="form#submit">Submit</button>
</div>
```

Targets are scoped to the controller, named semantically, and survive CSS refactors. They throw clear errors when missing instead of silently returning null.

---

## 5. Hardcoded Selectors

**Problem:** Hardcoding CSS classes, IDs, or attribute values in JavaScript. When the HTML changes, the controller breaks silently.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  toggle() {
    this.element.querySelector(".dropdown-menu").classList.toggle("is-open")
    this.element.querySelector(".dropdown-icon").classList.toggle("rotate-180")
    this.element.querySelector(".dropdown-backdrop").classList.toggle("opacity-50")
  }
}
```

**Corrected:**

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu", "icon", "backdrop"]
  static classes = ["open", "rotated", "visible"]
  static values = { open: { type: Boolean, default: false } }

  openValueChanged() {
    this.menuTarget.classList.toggle(this.openClass, this.openValue)
    this.iconTarget.classList.toggle(this.rotatedClass, this.openValue)

    if (this.hasBackdropTarget) {
      this.backdropTarget.classList.toggle(this.visibleClass, this.openValue)
    }
  }

  toggle() {
    this.openValue = !this.openValue
  }
}
```

```html
<div data-controller="dropdown"
     data-dropdown-open-class="is-open"
     data-dropdown-rotated-class="rotate-180"
     data-dropdown-visible-class="opacity-50">
  <button data-action="dropdown#toggle">
    <span data-dropdown-target="icon">▶</span>
    Menu
  </button>
  <ul data-dropdown-target="menu">...</ul>
  <div data-dropdown-target="backdrop"></div>
</div>
```

The CSS Classes API maps logical names to real CSS classes in HTML, making the controller reusable across different styling frameworks.

---

## 6. Business Logic in Controllers

**Problem:** Putting validation rules, calculations, or data transformations directly in the controller. Controllers should bridge HTML and behavior, not contain business logic.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  calculate() {
    const subtotal = this.itemTargets.reduce((sum, item) => {
      const price = parseFloat(item.dataset.price)
      const quantity = parseInt(item.querySelector("input").value, 10)
      return sum + (price * quantity)
    }, 0)

    // Tax calculation embedded in controller
    const taxRate = this.element.dataset.state === "CA" ? 0.0725 :
                    this.element.dataset.state === "NY" ? 0.08 :
                    this.element.dataset.state === "TX" ? 0.0625 : 0
    const tax = subtotal * taxRate

    // Discount logic embedded in controller
    let discount = 0
    if (subtotal > 100) discount = subtotal * 0.1
    if (subtotal > 500) discount = subtotal * 0.15

    const total = subtotal + tax - discount
    this.totalTarget.textContent = `$${total.toFixed(2)}`
  }
}
```

**Corrected:**

```javascript
// app/javascript/utils/pricing.js
export function calculateTotal({ subtotal, state, discountRules }) {
  const tax = subtotal * taxRateFor(state)
  const discount = applyDiscount(subtotal, discountRules)
  return subtotal + tax - discount
}

function taxRateFor(state) {
  const rates = { CA: 0.0725, NY: 0.08, TX: 0.0625 }
  return rates[state] || 0
}

function applyDiscount(subtotal, rules) {
  for (const rule of rules.sort((a, b) => b.threshold - a.threshold)) {
    if (subtotal >= rule.threshold) return subtotal * rule.rate
  }
  return 0
}
```

```javascript
// app/javascript/controllers/cart_controller.js
import { Controller } from "@hotwired/stimulus"
import { calculateTotal } from "../utils/pricing"

export default class extends Controller {
  static targets = ["item", "total"]
  static values = {
    state: String,
    discountRules: { type: Array, default: [] }
  }

  calculate() {
    const subtotal = this.itemTargets.reduce((sum, item) => {
      return sum + parseFloat(item.dataset.lineTotal)
    }, 0)

    const total = calculateTotal({
      subtotal,
      state: this.stateValue,
      discountRules: this.discountRulesValue
    })

    this.totalTarget.textContent = `$${total.toFixed(2)}`
  }
}
```

Business logic lives in a testable, reusable utility module. The controller only bridges DOM and logic. Even better: calculate the total server-side and return it via Turbo.

---

## 7. Tightly Coupled Controllers

**Problem:** Controllers reference each other by querying the DOM for specific controllers, creating invisible dependencies that break when the HTML structure changes.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  submit() {
    // Reaching into the DOM for another controller
    const notificationEl = document.querySelector("[data-controller='notification']")
    const notificationController = this.application.getControllerForElementAndIdentifier(
      notificationEl, "notification"
    )
    notificationController.show("Form submitted!")

    // Or even worse: direct DOM manipulation of another controller's elements
    document.querySelector("#notification-text").textContent = "Form submitted!"
    document.querySelector("#notification-container").classList.remove("hidden")
  }
}
```

**Corrected (with outlets):**

```javascript
// form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["notification"]

  submit() {
    // Type-safe outlet reference
    if (this.hasNotificationOutlet) {
      this.notificationOutlet.show("Form submitted!")
    }
  }
}
```

```html
<div data-controller="form"
     data-form-notification-outlet="#notifications">
  <button data-action="form#submit">Submit</button>
</div>

<div data-controller="notification" id="notifications">...</div>
```

**Corrected (with events for loose coupling):**

```javascript
// form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  submit() {
    // Dispatch event -- any controller can listen
    this.dispatch("submitted", { detail: { message: "Form submitted!" } })
  }
}

// notification_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  show(event) {
    this.element.textContent = event.detail.message
    this.element.hidden = false
  }
}
```

```html
<div data-controller="form"
     data-action="form:submitted->notification#show">
  <button data-action="form#submit">Submit</button>

  <div data-controller="notification" hidden></div>
</div>
```

Outlets for known relationships, events for loose coupling. Never use `getControllerForElementAndIdentifier` or global DOM queries.

---

## 8. Ignoring Action Parameters

**Problem:** Using generic `data-*` attributes and manually parsing them in event handlers instead of using Stimulus action parameters for typed, scoped data passing.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  remove(event) {
    const id = parseInt(event.target.closest("button").dataset.id, 10)
    const name = event.target.closest("button").dataset.name
    const confirmed = event.target.closest("button").dataset.confirm === "true"

    if (confirmed || confirm(`Delete ${name}?`)) {
      this.deleteItem(id)
    }
  }
}
```

```html
<button data-action="list#remove" data-id="42" data-name="Widget" data-confirm="false">
  Delete
</button>
```

**Corrected:**

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  remove(event) {
    // Typed parameters, no manual parsing needed
    const { id, name, confirm: needsConfirm } = event.params

    if (!needsConfirm || confirm(`Delete ${name}?`)) {
      this.deleteItem(id)
    }
  }

  deleteItem(id) {
    // ...
  }
}
```

```html
<button data-action="list#remove"
        data-list-id-param="42"
        data-list-name-param="Widget"
        data-list-confirm-param="false">
  Delete
</button>
```

Action parameters are automatically typed (42 becomes a Number, false becomes a Boolean), scoped to the controller (no collisions), and do not require `.closest()` traversal.

---

## 9. Not Using CSS Classes API

**Problem:** Hardcoding CSS class names in JavaScript. When the design system changes class names, the controller breaks.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  activate(event) {
    this.element.querySelectorAll(".tab").forEach(tab => {
      tab.classList.remove("bg-blue-500", "text-white", "font-bold")
      tab.classList.add("bg-gray-200", "text-gray-600")
    })

    event.currentTarget.classList.remove("bg-gray-200", "text-gray-600")
    event.currentTarget.classList.add("bg-blue-500", "text-white", "font-bold")
  }
}
```

**Corrected:**

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["tab"]
  static classes = ["active", "inactive"]
  static values = { index: { type: Number, default: 0 } }

  indexValueChanged() {
    this.tabTargets.forEach((tab, i) => {
      if (i === this.indexValue) {
        tab.classList.add(...this.activeClasses)
        tab.classList.remove(...this.inactiveClasses)
      } else {
        tab.classList.remove(...this.activeClasses)
        tab.classList.add(...this.inactiveClasses)
      }
    })
  }

  select(event) {
    this.indexValue = this.tabTargets.indexOf(event.currentTarget)
  }
}
```

```html
<div data-controller="tabs"
     data-tabs-active-class="bg-blue-500 text-white font-bold"
     data-tabs-inactive-class="bg-gray-200 text-gray-600">
  <button data-tabs-target="tab" data-action="tabs#select">Tab 1</button>
  <button data-tabs-target="tab" data-action="tabs#select">Tab 2</button>
  <button data-tabs-target="tab" data-action="tabs#select">Tab 3</button>
</div>
```

The CSS Classes API maps logical class names to real CSS classes in HTML. When the design system changes from Tailwind to Bootstrap, only the HTML attributes change -- the JavaScript remains untouched.

**Note:** `this.activeClasses` (plural) returns an array when the attribute contains space-separated classes. Use the spread operator (`...`) to pass them to `classList.add()`/`classList.remove()`.

---

## 10. Synchronous Heavy Operations in connect

**Problem:** Running expensive synchronous operations in `connect()` that block the main thread and degrade INP.

**Bad:**

```javascript
import { Controller } from "@hotwired/stimulus"
import { marked } from "marked"
import hljs from "highlight.js"

// DO NOT DO THIS
export default class extends Controller {
  connect() {
    // Synchronous: blocks rendering for every code block on the page
    this.element.querySelectorAll("pre code").forEach(block => {
      hljs.highlightElement(block) // Can take 50-200ms per block
    })

    // Synchronous: blocks rendering for large markdown content
    this.element.querySelectorAll(".markdown").forEach(el => {
      el.innerHTML = marked(el.textContent) // Can take 100ms+ for large docs
    })
  }
}
```

**Corrected:**

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["code"]

  connect() {
    this.processQueue = [...this.codeTargets]
    this.processNext()
  }

  disconnect() {
    this.processQueue = []
    if (this.frameId) cancelAnimationFrame(this.frameId)
  }

  processNext() {
    if (this.processQueue.length === 0) return

    const block = this.processQueue.shift()

    // Process one block per animation frame to avoid blocking
    this.frameId = requestAnimationFrame(async () => {
      const hljs = await import("highlight.js")
      hljs.default.highlightElement(block)
      this.processNext()
    })
  }
}
```

Heavy operations are chunked across animation frames so the browser can handle user interactions between each chunk. The heavy library is dynamically imported to avoid loading it on pages that do not need it.
