---
title: "Outlets API"
skill: stimulus-controllers
topic: Stimulus Outlets for controller-to-controller communication
---

# Outlets API

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Declaring Outlets](#declaring-outlets)
  - [Outlet Properties](#outlet-properties)
  - [Outlet Callbacks](#outlet-callbacks)
  - [Bidirectional Communication](#bidirectional-communication)
  - [Outlets vs Events vs Globals](#outlets-vs-events-vs-globals)
- [Pattern Card](#pattern-card)

## Overview

Outlets connect one Stimulus controller to another via HTML attributes. They provide typed, declarative references between controllers without relying on DOM structure, global state, or event buses.

For each declared outlet, Stimulus generates:

| Property | Type | Description |
|----------|------|-------------|
| `this.nameOutlet` | `Controller` | The first matching outlet controller instance (throws if missing) |
| `this.nameOutlets` | `Controller[]` | All matching outlet controller instances |
| `this.hasNameOutlet` | `Boolean` | Whether at least one outlet is connected |
| `this.nameOutletElement` | `Element` | The DOM element of the first matching outlet |
| `this.nameOutletElements` | `Element[]` | The DOM elements of all matching outlets |

And two callbacks:

| Callback | When It Fires |
|----------|---------------|
| `nameOutletConnected(controller, element)` | When an outlet controller connects |
| `nameOutletDisconnected(controller, element)` | When an outlet controller disconnects |

**When to use outlets:**

- One controller needs to call methods on another specific controller.
- The relationship is known at HTML authoring time.
- You need type safety (outlet references are actual controller instances with methods).

## Implementation

### Declaring Outlets

Outlets are declared as a static array and connected via `data-*-*-outlet` attributes pointing to a CSS selector:

```javascript
// app/javascript/controllers/search_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["search-results"]
  static values = { query: String }

  filter() {
    this.searchResultsOutlet.update(this.queryValue)
  }
}
```

```javascript
// app/javascript/controllers/search_results_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["item"]

  update(query) {
    this.itemTargets.forEach(item => {
      item.hidden = !item.textContent.toLowerCase().includes(query.toLowerCase())
    })
  }
}
```

```html
<div data-controller="search"
     data-search-search-results-outlet="#results">
  <input type="search"
         data-search-target="input"
         data-action="input->search#filter">
</div>

<div data-controller="search-results" id="results">
  <div data-search-results-target="item">Ruby on Rails</div>
  <div data-search-results-target="item">Stimulus</div>
  <div data-search-results-target="item">Turbo</div>
</div>
```

The outlet attribute `data-search-search-results-outlet="#results"` follows the pattern: `data-[source-controller]-[outlet-name]-outlet="[CSS selector]"`.

### Outlet Properties

Access outlets using the generated properties:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["player"]

  playAll() {
    // Call a method on all connected player outlets
    this.playerOutlets.forEach(player => player.play())
  }

  pauseFirst() {
    // Guard optional outlets
    if (this.hasPlayerOutlet) {
      this.playerOutlet.pause()
    }
  }

  getElements() {
    // Access the outlet's DOM element
    this.playerOutletElements.forEach(element => {
      element.classList.add("active")
    })
  }
}
```

**Important:** `this.nameOutlet` throws if no matching outlet exists. Always use `this.hasNameOutlet` when the outlet may be absent:

```javascript
// SAFE
if (this.hasPlayerOutlet) {
  this.playerOutlet.play()
}

// UNSAFE: throws if no player outlet is connected
this.playerOutlet.play()
```

### Outlet Callbacks

Outlet callbacks fire when outlet controllers connect or disconnect. This is useful for initialization that depends on another controller being ready:

```javascript
// app/javascript/controllers/form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["validation"]

  validationOutletConnected(controller, element) {
    // The validation controller is now ready
    controller.configure({
      rules: this.validationRules(),
      onValid: () => this.enableSubmit(),
      onInvalid: () => this.disableSubmit()
    })
  }

  validationOutletDisconnected(controller, element) {
    this.disableSubmit()
  }

  validationRules() {
    return {
      email: { required: true, format: "email" },
      name: { required: true, minLength: 2 }
    }
  }

  enableSubmit() {
    this.element.querySelector("[type=submit]").disabled = false
  }

  disableSubmit() {
    this.element.querySelector("[type=submit]").disabled = true
  }
}
```

### Bidirectional Communication

Two controllers can reference each other via outlets. This creates a tight, explicit coupling:

```javascript
// app/javascript/controllers/sidebar_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["main-content"]
  static values = { open: { type: Boolean, default: true } }

  openValueChanged() {
    this.element.classList.toggle("sidebar--collapsed", !this.openValue)

    if (this.hasMainContentOutlet) {
      this.mainContentOutlet.sidebarToggled(this.openValue)
    }
  }

  toggle() {
    this.openValue = !this.openValue
  }
}
```

```javascript
// app/javascript/controllers/main_content_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["sidebar"]

  sidebarToggled(isOpen) {
    this.element.classList.toggle("main-content--full", !isOpen)
  }

  expandSidebar() {
    if (this.hasSidebarOutlet) {
      this.sidebarOutlet.openValue = true
    }
  }
}
```

```html
<div data-controller="sidebar"
     data-sidebar-main-content-outlet="#main"
     data-sidebar-open-value="true">
  <button data-action="sidebar#toggle">Toggle Sidebar</button>
  <!-- sidebar content -->
</div>

<div data-controller="main-content"
     data-main-content-sidebar-outlet="[data-controller='sidebar']"
     id="main">
  <!-- main content -->
</div>
```

**Caution:** Bidirectional outlets create tight coupling. Prefer unidirectional outlets or events when the relationship does not require direct method calls in both directions.

### Outlets vs Events vs Globals

| Approach | Coupling | Type Safety | Use When |
|----------|----------|-------------|----------|
| **Outlets** | Explicit, declared | Yes (controller instances) | Known, direct relationship between two controllers |
| **Custom events** (`this.dispatch()`) | Loose | No (event.detail) | Broadcasting to unknown listeners |
| **Global state** (window, document) | None | No | Never in Stimulus (use outlets or events) |

```javascript
// Outlets: direct method call
this.searchResultsOutlet.update(query)

// Events: broadcast, anyone can listen
this.dispatch("searched", { detail: { query } })
// Listener: data-action="search:searched->results#handleSearch"

// Global: DO NOT DO THIS
window.searchQuery = query
```

**Choose outlets when:**
- You know exactly which controller needs to respond.
- You need to call specific methods with arguments.
- The relationship is visible in the HTML.

**Choose events when:**
- Multiple controllers might respond.
- The responding controllers are not known at authoring time.
- You want the loosest possible coupling.

## Pattern Card

### GOOD: Outlet-based coordination

```javascript
// Dropdown controller tells the tooltip controller to hide
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["tooltip"]
  static values = { open: { type: Boolean, default: false } }

  toggle() {
    this.openValue = !this.openValue
  }

  openValueChanged() {
    // Direct, typed communication
    if (this.openValue && this.hasTooltipOutlet) {
      this.tooltipOutlet.hide()
    }
  }
}
```

```html
<div data-controller="dropdown"
     data-dropdown-tooltip-outlet="#user-tooltip">
  <button data-action="dropdown#toggle">Menu</button>
</div>

<div data-controller="tooltip" id="user-tooltip">
  Helpful tooltip
</div>
```

The relationship is explicit in HTML, type-safe in JavaScript, and easy to trace during debugging.

### BAD: Global event bus or window variables

```javascript
// DO NOT DO THIS
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  toggle() {
    this.open = !this.open

    // Globals are untraceable and pollute the shared namespace
    window.dropdownState = { open: this.open }

    // Event bus with no clear consumer
    document.dispatchEvent(
      new CustomEvent("dropdown:toggled", { detail: { open: this.open } })
    )
  }
}
```

Global variables create hidden dependencies. Custom events on `document` make it impossible to trace who is listening. The relationship between controllers is invisible in the HTML and requires reading every controller's source to understand.
