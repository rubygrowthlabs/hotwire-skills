---
title: "Action Parameters and Keyboard Events"
skill: stimulus-controllers
topic: Stimulus action parameters, keyboard event filters, and action options
---

# Action Parameters and Keyboard Events

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Action Parameters](#action-parameters)
  - [Action Parameter Types](#action-parameter-types)
  - [Keyboard Event Filters](#keyboard-event-filters)
  - [Action Options](#action-options)
  - [Action Descriptor Syntax Reference](#action-descriptor-syntax-reference)
  - [Combining Parameters, Filters, and Options](#combining-parameters-filters-and-options)
- [Pattern Card](#pattern-card)

## Overview

Stimulus action descriptors are the `data-action` attributes that wire DOM events to controller methods. They support three powerful features beyond basic event binding:

1. **Action parameters**: Pass typed data from HTML attributes directly to the action handler via `event.params`.
2. **Keyboard event filters**: Restrict keyboard actions to specific keys (e.g., only fire on Enter or Escape).
3. **Action options**: Modify event handling behavior (prevent default, stop propagation, etc.).

These features eliminate the need for manual `event.key` checking, `event.preventDefault()` calls, and `dataset` parsing.

## Implementation

### Action Parameters

Action parameters pass data from HTML to JavaScript without manually reading `dataset` attributes. Parameters are declared as `data-[controller]-[name]-param` attributes on the element with the action:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  add(event) {
    // Parameters are available on event.params
    const { id, quantity, name } = event.params
    console.log(`Adding ${quantity}x ${name} (ID: ${id})`)
  }
}
```

```html
<div data-controller="cart">
  <button data-action="cart#add"
          data-cart-id-param="42"
          data-cart-quantity-param="1"
          data-cart-name-param="Widget">
    Add to Cart
  </button>

  <button data-action="cart#add"
          data-cart-id-param="99"
          data-cart-quantity-param="3"
          data-cart-name-param="Gadget">
    Add 3 Gadgets
  </button>
</div>
```

**Key difference from dataset:** Action parameters are scoped to the specific action and controller. They use the `data-[controller]-[name]-param` naming convention, not generic `data-*` attributes.

### Action Parameter Types

Stimulus automatically coerces action parameters based on the value:

| HTML Value | Coerced Type | JavaScript Value |
|-----------|-------------|-----------------|
| `"42"` | Number | `42` |
| `"3.14"` | Number | `3.14` |
| `"true"` | Boolean | `true` |
| `"false"` | Boolean | `false` |
| `'{"a":1}'` | Object | `{ a: 1 }` |
| `'[1,2,3]'` | Array | `[1, 2, 3]` |
| `"hello"` | String | `"hello"` |

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  configure(event) {
    const { limit, enabled, options } = event.params
    console.log(typeof limit)    // "number"
    console.log(typeof enabled)  // "boolean"
    console.log(typeof options)  // "object"
  }
}
```

```html
<button data-action="settings#configure"
        data-settings-limit-param="100"
        data-settings-enabled-param="true"
        data-settings-options-param='{"theme":"dark","lang":"en"}'>
  Apply Settings
</button>
```

### Keyboard Event Filters

Keyboard filters restrict when an action fires based on the key pressed. Add the key name after the event type, separated by a dot:

```html
<!-- Only fires on Enter key -->
<input data-action="keydown.enter->search#submit">

<!-- Only fires on Escape key -->
<div data-action="keydown.esc->modal#close">

<!-- Only fires on Arrow Down -->
<input data-action="keydown.down->autocomplete#next">

<!-- Multiple key filters (separate actions) -->
<input data-action="keydown.up->autocomplete#previous keydown.down->autocomplete#next keydown.enter->autocomplete#select keydown.esc->autocomplete#dismiss">
```

**Available key filters:**

| Filter | Key |
|--------|-----|
| `.enter` | Enter |
| `.tab` | Tab |
| `.esc` | Escape |
| `.space` | Space |
| `.up` | ArrowUp |
| `.down` | ArrowDown |
| `.left` | ArrowLeft |
| `.right` | ArrowRight |
| `.home` | Home |
| `.end` | End |
| `.page_up` | PageUp |
| `.page_down` | PageDown |
| `.[key]` | Any `KeyboardEvent.key` value |

You can also use exact `KeyboardEvent.key` values as filters:

```html
<!-- These are equivalent -->
<input data-action="keydown.enter->form#submit">
<input data-action="keydown.Enter->form#submit">

<!-- Letter keys -->
<div data-action="keydown.k->search#focus">

<!-- Slash key for search focus (common pattern) -->
<body data-action="keydown./->search#focus">
```

### Action Options

Action options modify how the event is handled. Append them to the action descriptor with a colon:

| Option | Effect | Equivalent To |
|--------|--------|---------------|
| `:prevent` | Calls `event.preventDefault()` | `event.preventDefault()` in handler |
| `:stop` | Calls `event.stopPropagation()` | `event.stopPropagation()` in handler |
| `:self` | Only fires if `event.target` is the element itself | `if (event.target !== this.element) return` |
| `:once` | Removes the listener after first invocation | `{ once: true }` option |
| `:capture` | Uses capture phase | `{ capture: true }` option |
| `:passive` | Marks listener as passive | `{ passive: true }` option |

```html
<!-- Prevent form submission default behavior -->
<form data-action="submit->checkout#process:prevent">

<!-- Stop event propagation -->
<button data-action="click->dropdown#toggle:stop">Menu</button>

<!-- Only fire if clicking the backdrop itself, not its children -->
<div data-action="click->modal#close:self" class="modal-backdrop">
  <div class="modal-content">
    <!-- Clicks here do NOT trigger close -->
  </div>
</div>

<!-- Fire only once (e.g., analytics tracking) -->
<button data-action="click->analytics#trackFirstClick:once">
  Get Started
</button>

<!-- Combine options -->
<form data-action="submit->form#save:prevent:stop">
```

### Action Descriptor Syntax Reference

The full syntax of an action descriptor is:

```
event->controller#method:option1:option2
```

With keyboard filters:

```
event.key->controller#method:option1:option2
```

**Default events** -- Stimulus infers the event for common elements:

| Element | Default Event |
|---------|--------------|
| `<input>` | `input` |
| `<select>` | `change` |
| `<textarea>` | `input` |
| `<form>` | `submit` |
| `<details>` | `toggle` |
| Everything else | `click` |

```html
<!-- These are equivalent -->
<form data-action="submit->form#save">
<form data-action="form#save">

<!-- These are equivalent -->
<input data-action="input->search#filter">
<input data-action="search#filter">

<!-- These are equivalent -->
<button data-action="click->menu#toggle">
<button data-action="menu#toggle">
```

### Combining Parameters, Filters, and Options

All three features work together:

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["results"]

  navigate(event) {
    const { url, method } = event.params
    // event.preventDefault() already called by :prevent
    // Event only fires on Enter key thanks to .enter filter

    if (method === "turbo") {
      Turbo.visit(url)
    } else {
      window.location.href = url
    }
  }
}
```

```html
<div data-controller="nav">
  <!-- On Enter: navigate to URL, prevent default, stop propagation -->
  <input data-action="keydown.enter->nav#navigate:prevent:stop"
         data-nav-url-param="/dashboard"
         data-nav-method-param="turbo"
         placeholder="Press Enter to go to Dashboard">
</div>
```

A practical autocomplete example combining all features:

```html
<div data-controller="autocomplete">
  <input data-autocomplete-target="input"
         data-action="
           input->autocomplete#search
           keydown.down->autocomplete#next:prevent
           keydown.up->autocomplete#previous:prevent
           keydown.enter->autocomplete#select:prevent
           keydown.esc->autocomplete#dismiss
         ">

  <ul data-autocomplete-target="results" hidden>
    <!-- results populated dynamically -->
  </ul>
</div>
```

## Pattern Card

### GOOD: Action parameters with keyboard filter

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["output"]

  select(event) {
    // Typed parameters, no manual parsing
    const { id, label } = event.params
    this.outputTarget.textContent = `Selected: ${label} (${id})`
  }
}
```

```html
<div data-controller="picker">
  <p data-picker-target="output">Nothing selected</p>

  <!-- Keyboard filter handles key detection -->
  <button data-action="click->picker#select keydown.enter->picker#select"
          data-picker-id-param="1"
          data-picker-label-param="Ruby">
    Ruby
  </button>

  <button data-action="click->picker#select keydown.enter->picker#select"
          data-picker-id-param="2"
          data-picker-label-param="Python">
    Python
  </button>
</div>
```

Parameters are declarative, typed, and scoped to the controller. Keyboard filtering is handled by Stimulus, not by manual `event.key` checks.

### BAD: Manual event.key checking and dataset parsing

```javascript
import { Controller } from "@hotwired/stimulus"

// DO NOT DO THIS
export default class extends Controller {
  select(event) {
    // Manual key checking -- verbose and error-prone
    if (event.type === "keydown" && event.key !== "Enter") return

    // Manual dataset parsing -- no type safety, fragile naming
    const id = parseInt(event.target.dataset.id, 10)
    const label = event.target.dataset.label

    this.element.querySelector(".output").textContent = `Selected: ${label} (${id})`
  }
}
```

```html
<div data-controller="picker">
  <p class="output">Nothing selected</p>

  <!-- No keyboard filter, relies on manual checking -->
  <button data-action="click->picker#select keydown->picker#select"
          data-id="1"
          data-label="Ruby">
    Ruby
  </button>
</div>
```

Manual key checking is repeated in every handler. Dataset parsing has no type coercion and uses generic `data-*` attributes that can collide with other libraries. The `querySelector` call is fragile and not scoped to the controller.
