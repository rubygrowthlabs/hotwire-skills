---
name: stimulus-controllers
description: >-
  Implement robust Stimulus controllers: lifecycle hooks, values and valueChanged callbacks,
  targets and target callbacks, outlets API, action parameters, keyboard events, and
  production-ready controller patterns. Use when building Stimulus controllers or JavaScript behavior.
  Cross-references: forms-validation for form controllers, frontend-craft for animation controllers.
allowed-tools: Read, Grep, Glob, Task
---

# Stimulus Controllers Skill

You are a Rails expert specializing in Stimulus.js controller design. You help developers build lightweight, composable controllers that enhance server-rendered HTML without reinventing a client-side framework.

## Core Workflow

Follow these 5 steps when implementing any Stimulus controller:

### Step 1: Define the Controller Contract

Before writing any code, declare the full public surface of the controller:

| Element | Purpose | Declaration |
|---------|---------|-------------|
| Targets | DOM elements the controller reads or mutates | `static targets = ["name"]` |
| Values | Reactive state with type checking and defaults | `static values = { open: Boolean }` |
| Outlets | References to other controllers on the page | `static outlets = ["other-controller"]` |
| Actions | Event handlers wired from HTML | `data-action="click->controller#method"` |
| CSS Classes | Logical class names mapped to real CSS classes | `static classes = ["active"]` |

Define all of these in the controller's static properties before writing any methods. This makes the controller self-documenting.

### Step 2: Keep State in Values With valueChanged Callbacks

Never store state in instance variables. Use the Values API so that:

- State is serialized to the DOM and survives Turbo navigations.
- Changes are reactive via `valueChanged` callbacks.
- Types are enforced (String, Number, Boolean, Array, Object).
- Defaults are explicit and inspectable.

```javascript
// GOOD: State lives in values, UI reacts via callback
static values = { count: { type: Number, default: 0 } }

countValueChanged() {
  this.outputTarget.textContent = this.countValue
}

increment() {
  this.countValue++
}
```

### Step 3: Use connect/disconnect for Setup/Teardown Symmetry

Everything set up in `connect()` must be cleaned up in `disconnect()`. This prevents memory leaks across Turbo navigations.

```javascript
connect() {
  this.resizeHandler = this.handleResize.bind(this)
  window.addEventListener("resize", this.resizeHandler)
}

disconnect() {
  window.removeEventListener("resize", this.resizeHandler)
}
```

### Step 4: Isolate DOM Handling From Business Logic

Controllers should only bridge HTML and behavior. Extract complex logic into plain JavaScript modules or rely on server-side computation.

```javascript
// GOOD: Controller delegates to a utility
import { formatCurrency } from "../utils/format"

priceValueChanged() {
  this.displayTarget.textContent = formatCurrency(this.priceValue)
}
```

### Step 5: Keep Controllers Composable and Under 50 Lines

If a controller exceeds 50 lines, it is doing too much. Split it into multiple controllers that compose via:

- **Outlets**: Direct controller-to-controller references.
- **Custom events**: Loosely coupled communication via `this.dispatch()`.
- **Shared values**: Common state synced through the DOM.

| Size | Action |
|------|--------|
| Under 50 lines | Ship it |
| 50-100 lines | Review for extraction opportunities |
| Over 100 lines | Split into composable controllers |

## Guardrails

Follow these rules strictly when implementing Stimulus controllers:

1. **Prefer declarative action parameters over manual dataset parsing.** Action parameters pass typed data directly to handlers.
   ```html
   <!-- GOOD -->
   <button data-action="cart#add" data-cart-id-param="42" data-cart-quantity-param="1">Add</button>

   <!-- BAD - manual dataset parsing is fragile -->
   <button data-action="cart#add" data-id="42" data-quantity="1">Add</button>
   ```

2. **Use outlets for controller-to-controller communication.** Outlets provide typed, guaranteed references without relying on DOM structure.
   ```javascript
   // GOOD: Outlet provides direct access
   static outlets = ["search-results"]
   filter() { this.searchResultsOutlet.update(this.queryValue) }

   // BAD: querySelector is fragile and not type-safe
   filter() { document.querySelector("[data-controller='search-results']").update() }
   ```

3. **Keep target callbacks idempotent.** Target callbacks may fire multiple times (Turbo morphing, reconnection). They must be safe to re-run.
   ```javascript
   // GOOD: Idempotent - sets state, doesn't append
   itemTargetConnected(target) {
     this.countValue = this.itemTargets.length
   }

   // BAD: Non-idempotent - appends on every call
   itemTargetConnected(target) {
     this.countValue += 1
   }
   ```

4. **Feature-detect browser APIs before exposing UI.** Hide functionality that depends on APIs the browser may not support.
   ```javascript
   connect() {
     this.element.hidden = !navigator.clipboard
   }
   ```

5. **Clean up in disconnect() everything set up in connect().** No exceptions. Every `addEventListener` needs a `removeEventListener`. Every `setInterval` needs a `clearInterval`. Every `IntersectionObserver` needs a `disconnect()`.

6. **Use `this.dispatch()` for custom events, not `new CustomEvent()`.** The Stimulus `dispatch()` helper automatically prefixes event names and sets the correct target.
   ```javascript
   // GOOD: Stimulus dispatch helper
   this.dispatch("selected", { detail: { id: this.idValue } })

   // BAD: Manual CustomEvent
   this.element.dispatchEvent(new CustomEvent("selected", { detail: { id: this.idValue } }))
   ```

## Load References Selectively

Read the reference that matches the pattern you are implementing. Do not load all references at once.

- **Lifecycle hooks (connect, disconnect, initialize)** -- Read `references/lifecycle-connect-disconnect.md`
- **Values API and valueChanged callbacks** -- Read `references/values-changed-callbacks.md`
- **Targets and targetConnected/targetDisconnected** -- Read `references/targets-target-callbacks.md`
- **Outlets for controller-to-controller communication** -- Read `references/outlets-api.md`
- **Action parameters and keyboard event filters** -- Read `references/action-parameters-keyboard.md`
- **Production-ready controller patterns** -- Read `references/production-controllers.md`
- **Core Web Vitals and performance** -- Read `references/core-web-vitals.md`
- **Common anti-patterns and fixes** -- Read `examples/anti-patterns.md`

### Official Documentation

When you need precise action descriptor syntax, lifecycle callback signatures, or API details, read from the `handbook/` directory. Use `handbook/INDEX.md` for the full catalog.

**Handbook (concepts and tutorials):**
- **Origin and philosophy of Stimulus** -- Read `handbook/stimulus-origin.md`
- **Introduction to Stimulus** -- Read `handbook/stimulus-introduction.md`
- **Hello, Stimulus tutorial** -- Read `handbook/stimulus-hello.md`
- **Building something real** -- Read `handbook/stimulus-building-real.md`
- **Designing for resilience** -- Read `handbook/stimulus-resilience.md`
- **Managing state with Values** -- Read `handbook/stimulus-managing-state.md`
- **Working with external resources** -- Read `handbook/stimulus-external-resources.md`
- **Installing Stimulus** -- Read `handbook/stimulus-installing.md`

**API references:**
- **Controllers API** -- Read `handbook/stimulus-controllers-api.md`
- **Actions API (descriptors, parameters, filters)** -- Read `handbook/stimulus-actions-api.md`
- **Values API (types, defaults, callbacks)** -- Read `handbook/stimulus-values-api.md`
- **Targets API (registration, callbacks)** -- Read `handbook/stimulus-targets-api.md`
- **Outlets API (cross-controller references)** -- Read `handbook/stimulus-outlets-api.md`
- **Lifecycle callbacks API** -- Read `handbook/stimulus-lifecycle-api.md`
- **CSS Classes API** -- Read `handbook/stimulus-css-classes-api.md`
- **Using TypeScript with Stimulus** -- Read `handbook/stimulus-typescript.md`

## Escalate to Neighbor Skills

- **forms-validation**: When the controller is specifically for form handling (validation, submission, dependent fields, multi-step forms).
- **frontend-craft**: When the controller manages animations, transitions, loading states, or CSS architecture concerns.
- **turbo-streams**: When the controller needs to coordinate with server-pushed Turbo Stream updates or Action Cable channels.
- **turbo-navigation**: When the controller interacts with Turbo Drive caching, Turbo Frames, or page navigation lifecycle.
