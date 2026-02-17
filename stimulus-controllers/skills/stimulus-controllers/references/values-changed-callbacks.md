---
title: "Values and valueChanged Callbacks"
skill: stimulus-controllers
topic: Stimulus Values API with reactive change callbacks
---

# Values and valueChanged Callbacks

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Declaring Values With Types and Defaults](#declaring-values-with-types-and-defaults)
  - [Type Coercion Rules](#type-coercion-rules)
  - [valueChanged Callbacks](#valuechanged-callbacks)
  - [Reactive UI Updates](#reactive-ui-updates)
  - [Values and Turbo Cache](#values-and-turbo-cache)
  - [Complex State With Object and Array Values](#complex-state-with-object-and-array-values)
- [Pattern Card](#pattern-card)

## Overview

The Values API is Stimulus's reactive state system. Values are:

- **Declared** as static properties with types and optional defaults.
- **Serialized** to `data-*-*-value` attributes on the controller's element.
- **Type-checked** automatically (String, Number, Boolean, Array, Object).
- **Reactive** through `*ValueChanged` callbacks that fire whenever a value changes.

Values replace instance variables for all controller state. Because values live in the DOM, they survive Turbo cache restores and are inspectable in browser DevTools.

| Type | Default value | HTML attribute example |
|------|--------------|----------------------|
| `String` | `""` | `data-search-query-value="hello"` |
| `Number` | `0` | `data-counter-count-value="5"` |
| `Boolean` | `false` | `data-toggle-open-value="true"` |
| `Array` | `[]` | `data-tags-selected-value='["a","b"]'` |
| `Object` | `{}` | `data-config-options-value='{"key":"val"}'` |

## Implementation

### Declaring Values With Types and Defaults

There are two syntaxes for declaring values:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  // Short syntax: type only (uses type's default)
  static values = {
    url: String,
    count: Number,
    open: Boolean
  }

  // Long syntax: type + explicit default
  static values = {
    url: { type: String, default: "/api/search" },
    count: { type: Number, default: 0 },
    open: { type: Boolean, default: false },
    items: { type: Array, default: [] },
    config: { type: Object, default: { debounce: 300 } }
  }
}
```

Use the long syntax whenever a non-zero/non-empty default makes sense. It documents intent and reduces boilerplate in the HTML.

### Type Coercion Rules

Stimulus automatically coerces HTML attribute strings to the declared type:

```javascript
static values = { count: Number, active: Boolean, tags: Array }
```

```html
<!-- These string attributes are coerced automatically -->
<div data-controller="example"
     data-example-count-value="42"
     data-example-active-value="true"
     data-example-tags-value='["ruby","rails"]'>
</div>
```

| Declared Type | Attribute String | Coerced Value |
|---------------|-----------------|---------------|
| Number | `"42"` | `42` |
| Number | `"3.14"` | `3.14` |
| Boolean | `"true"` | `true` |
| Boolean | `"false"` | `false` |
| Boolean | `""` | `false` |
| Array | `'["a","b"]'` | `["a", "b"]` |
| Object | `'{"key":"val"}'` | `{ key: "val" }` |

**Important:** When setting values from JavaScript, use the typed setter:

```javascript
// GOOD: Use the typed property
this.countValue = 42
this.activeValue = true
this.tagsValue = ["ruby", "rails"]

// BAD: Do not manipulate data attributes directly
this.element.dataset.exampleCountValue = "42"
```

### valueChanged Callbacks

For every declared value, Stimulus generates a `*ValueChanged` callback method name:

| Value Name | Callback Method |
|-----------|----------------|
| `count` | `countValueChanged()` |
| `open` | `openValueChanged()` |
| `selectedItems` | `selectedItemsValueChanged()` |

The callback receives the new value and the previous value as arguments:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    count: { type: Number, default: 0 },
    open: { type: Boolean, default: false }
  }

  countValueChanged(value, previousValue) {
    console.log(`count changed from ${previousValue} to ${value}`)
  }

  openValueChanged(value, previousValue) {
    console.log(`open changed from ${previousValue} to ${value}`)
  }
}
```

**Key behavior:**

- `valueChanged` fires on `connect()` with the initial value (previousValue is undefined).
- `valueChanged` fires whenever the value is set, even to the same value.
- `valueChanged` fires when the HTML attribute is modified externally (e.g., by Turbo morph).

### Reactive UI Updates

The primary use of `valueChanged` is keeping the DOM in sync with state:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["count", "items", "emptyState"]
  static values = {
    count: { type: Number, default: 0 },
    filter: { type: String, default: "" }
  }

  // React to count changes
  countValueChanged() {
    this.countTarget.textContent = this.countValue
    this.emptyStateTarget.hidden = this.countValue > 0
  }

  // React to filter changes
  filterValueChanged() {
    this.itemTargets.forEach(item => {
      const text = item.textContent.toLowerCase()
      item.hidden = !text.includes(this.filterValue.toLowerCase())
    })
  }

  add() {
    this.countValue++
  }

  search(event) {
    this.filterValue = event.target.value
  }
}
```

```html
<div data-controller="list" data-list-count-value="3">
  <span data-list-target="count">3</span> items

  <input type="search"
         data-action="input->list#search"
         placeholder="Filter...">

  <div data-list-target="emptyState" hidden>No items found</div>

  <ul>
    <li data-list-target="items">Item 1</li>
    <li data-list-target="items">Item 2</li>
    <li data-list-target="items">Item 3</li>
  </ul>

  <button data-action="list#add">Add item</button>
</div>
```

### Values and Turbo Cache

Values are stored as HTML attributes, which means they are included in Turbo's page cache snapshots. When a user navigates back, the cached page includes the last-set values.

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    page: { type: Number, default: 1 },
    loaded: { type: Boolean, default: false }
  }

  connect() {
    // On cache restore, loadedValue is true from the cached HTML.
    // Only fetch if this is a fresh page load.
    if (!this.loadedValue) {
      this.fetchPage()
    }
  }

  async fetchPage() {
    const response = await fetch(`/items?page=${this.pageValue}`)
    // ... update DOM
    this.loadedValue = true
  }

  nextPage() {
    this.pageValue++
    this.loadedValue = false
    this.fetchPage()
  }
}
```

**Caveat:** If you need to reset values when navigating away, clean them up in `disconnect()` or in a `turbo:before-cache` listener.

### Complex State With Object and Array Values

Array and Object values enable richer state, but use them sparingly. Stimulus performs a deep comparison to detect changes, which has a performance cost.

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["badge", "list"]
  static values = {
    selected: { type: Array, default: [] }
  }

  selectedValueChanged() {
    this.badgeTarget.textContent = this.selectedValue.length
    this.updateSelectionUI()
  }

  toggle(event) {
    const id = event.params.id
    const selected = [...this.selectedValue]

    const index = selected.indexOf(id)
    if (index === -1) {
      selected.push(id)
    } else {
      selected.splice(index, 1)
    }

    // Must assign a new array for change detection
    this.selectedValue = selected
  }

  updateSelectionUI() {
    this.listTarget.querySelectorAll("[data-id]").forEach(item => {
      const id = parseInt(item.dataset.id, 10)
      item.classList.toggle("selected", this.selectedValue.includes(id))
    })
  }
}
```

**Important:** Mutating an Array or Object in place does not trigger `valueChanged`. You must assign a new reference:

```javascript
// GOOD: New array triggers change detection
this.selectedValue = [...this.selectedValue, newItem]

// BAD: Mutation is invisible to Stimulus
this.selectedValue.push(newItem) // valueChanged will NOT fire
```

## Pattern Card

### GOOD: State in values with reactive callback

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["output", "submitButton"]
  static values = {
    dirty: { type: Boolean, default: false },
    charCount: { type: Number, default: 0 }
  }

  dirtyValueChanged() {
    this.submitButtonTarget.disabled = !this.dirtyValue
  }

  charCountValueChanged() {
    this.outputTarget.textContent = `${this.charCountValue} characters`
  }

  input(event) {
    this.dirtyValue = true
    this.charCountValue = event.target.value.length
  }

  save() {
    this.dirtyValue = false
  }
}
```

State is declared, typed, and reactive. The UI updates automatically when values change. State is inspectable in the DOM via data attributes.

### BAD: State in instance variables

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  connect() {
    this.dirty = false
    this.charCount = 0
  }

  input(event) {
    this.dirty = true
    this.charCount = event.target.value.length
    // Must manually update DOM everywhere state is used
    this.element.querySelector(".submit-btn").disabled = !this.dirty
    this.element.querySelector(".char-count").textContent = `${this.charCount} characters`
  }

  save() {
    this.dirty = false
    this.element.querySelector(".submit-btn").disabled = true
  }
}
```

State is invisible, not typed, lost on Turbo cache restore, and requires manual DOM updates scattered throughout the controller. Every place that reads state must know where to update it.
