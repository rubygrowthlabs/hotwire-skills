---
title: "Production-Ready Controllers"
skill: stimulus-controllers
topic: Battle-tested Stimulus controller patterns from 37signals
---

# Production-Ready Controllers

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [1. Clipboard Controller (25 lines)](#1-clipboard-controller-25-lines)
  - [2. Auto-Click Controller (7 lines)](#2-auto-click-controller-7-lines)
  - [3. Toggle-Class Controller (31 lines)](#3-toggle-class-controller-31-lines)
  - [4. Auto-Submit Controller (28 lines)](#4-auto-submit-controller-28-lines)
  - [5. Dialog Controller (45 lines)](#5-dialog-controller-45-lines)
  - [6. Local-Time Controller (40 lines)](#6-local-time-controller-40-lines)
- [Pattern Card](#pattern-card)

## Overview

These six controllers are patterns extracted from production Rails applications at 37signals (Basecamp, HEY). They demonstrate the Stimulus philosophy: small, composable controllers that enhance server-rendered HTML.

Each controller:

- Is under 50 lines of code.
- Has a single, focused responsibility.
- Uses the Values API for state.
- Handles connect/disconnect symmetry.
- Includes feature detection where appropriate.

These are not libraries to install -- they are patterns to adapt for your application.

## Implementation

### 1. Clipboard Controller (25 lines)

Copies text to the clipboard with visual feedback. Feature-detects the Clipboard API and hides the button if unavailable.

```javascript
// app/javascript/controllers/clipboard_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["source", "button"]
  static values = {
    successDuration: { type: Number, default: 2000 }
  }

  connect() {
    if (!navigator.clipboard) {
      this.buttonTarget.hidden = true
    }
  }

  async copy() {
    const text = this.sourceTarget.value || this.sourceTarget.textContent
    await navigator.clipboard.writeText(text.trim())
    this.showFeedback()
  }

  showFeedback() {
    const original = this.buttonTarget.textContent
    this.buttonTarget.textContent = "Copied!"
    this.buttonTarget.disabled = true

    setTimeout(() => {
      this.buttonTarget.textContent = original
      this.buttonTarget.disabled = false
    }, this.successDurationValue)
  }
}
```

**HTML usage:**

```html
<div data-controller="clipboard" data-clipboard-success-duration-value="1500">
  <input data-clipboard-target="source" value="https://example.com/invite/abc123" readonly>
  <button data-clipboard-target="button"
          data-action="clipboard#copy">
    Copy Link
  </button>
</div>
```

**How it works:** The controller reads text from the source target (works with both `<input>` values and text content), copies it using the Clipboard API, then temporarily changes the button text to "Copied!" for visual feedback. The feedback duration is configurable via a value.

---

### 2. Auto-Click Controller (7 lines)

Automatically clicks an element when the controller connects. Useful for triggering actions on page load (e.g., opening a modal from a redirect, auto-playing a video).

```javascript
// app/javascript/controllers/auto_click_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.element.click()
  }
}
```

**HTML usage:**

```html
<%# Auto-open a modal after redirect %>
<% if flash[:open_modal] %>
  <button data-controller="auto-click"
          data-action="click->modal#open"
          hidden>
    Open Modal
  </button>
<% end %>

<%# Auto-submit a form (e.g., for SSO redirect) %>
<form data-controller="auto-click" data-action="click->form#submit" hidden>
  <input type="hidden" name="token" value="<%= @sso_token %>">
</form>
```

**How it works:** The controller calls `click()` on its element as soon as it connects. The element's own click handlers (including Stimulus actions on the same element) respond to the programmatic click. The element is typically `hidden` since it is just a trigger.

---

### 3. Toggle-Class Controller (31 lines)

A general-purpose class toggling controller driven by a boolean value. Supports toggling on the controller element or on a specific target.

```javascript
// app/javascript/controllers/toggle_class_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["toggleable"]
  static classes = ["toggle"]
  static values = {
    open: { type: Boolean, default: false }
  }

  openValueChanged() {
    this.toggleTargets()
  }

  toggle() {
    this.openValue = !this.openValue
  }

  show() {
    this.openValue = true
  }

  hide() {
    this.openValue = false
  }

  toggleTargets() {
    const targets = this.hasToggleableTarget
      ? this.toggleableTargets
      : [this.element]

    targets.forEach(target => {
      target.classList.toggle(this.toggleClass, this.openValue)
    })
  }
}
```

**HTML usage:**

```html
<%# Simple dropdown %>
<div data-controller="toggle-class"
     data-toggle-class-toggle-class="hidden"
     data-toggle-class-open-value="false">
  <button data-action="toggle-class#toggle">Menu</button>

  <ul data-toggle-class-target="toggleable" class="hidden">
    <li>Profile</li>
    <li>Settings</li>
    <li>Sign out</li>
  </ul>
</div>

<%# Sidebar collapse %>
<div data-controller="toggle-class"
     data-toggle-class-toggle-class="sidebar--collapsed"
     data-toggle-class-open-value="true">
  <button data-action="toggle-class#toggle">Collapse</button>
  <nav data-toggle-class-target="toggleable">
    <!-- sidebar content -->
  </nav>
</div>
```

**How it works:** The `openValue` drives the class state through `openValueChanged`. The CSS Classes API (`static classes`) maps the logical name "toggle" to any real CSS class specified in HTML. This makes the controller reusable across different styling frameworks.

---

### 4. Auto-Submit Controller (28 lines)

Debounced form submission triggered by input changes. Useful for search forms, filters, and auto-saving.

```javascript
// app/javascript/controllers/auto_submit_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    delay: { type: Number, default: 300 }
  }

  connect() {
    this.timeout = null
  }

  submit() {
    this.cancelPending()
    this.element.requestSubmit()
  }

  debouncedSubmit() {
    this.cancelPending()
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }

  cancelPending() {
    if (this.timeout) {
      clearTimeout(this.timeout)
      this.timeout = null
    }
  }

  disconnect() {
    this.cancelPending()
  }
}
```

**HTML usage:**

```html
<%# Search form with debounced auto-submit %>
<%= form_with url: search_path, method: :get,
    data: { controller: "auto-submit", auto_submit_delay_value: 500 } do |f| %>
  <%= f.search_field :q,
      data: { action: "input->auto-submit#debouncedSubmit" },
      placeholder: "Search..." %>
<% end %>

<%# Filter form with immediate submit on select change %>
<%= form_with url: products_path, method: :get,
    data: { controller: "auto-submit" } do |f| %>
  <%= f.select :category, ["All", "Books", "Electronics"],
      {}, data: { action: "change->auto-submit#submit" } %>
  <%= f.select :sort, ["Newest", "Price: Low", "Price: High"],
      {}, data: { action: "change->auto-submit#submit" } %>
<% end %>
```

**How it works:** The controller provides two submission methods: `submit()` for immediate submission (good for `<select>` changes) and `debouncedSubmit()` for text inputs where you want to wait for the user to stop typing. `requestSubmit()` is used instead of `submit()` to trigger Turbo form handling and HTML validation.

---

### 5. Dialog Controller (45 lines)

Manages native HTML `<dialog>` elements with proper modal/non-modal support, backdrop clicks, and keyboard handling.

```javascript
// app/javascript/controllers/dialog_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dialog"]
  static values = {
    open: { type: Boolean, default: false },
    modal: { type: Boolean, default: true }
  }

  connect() {
    this.boundHandleBackdropClick = this.handleBackdropClick.bind(this)
    this.dialogTarget.addEventListener("click", this.boundHandleBackdropClick)

    if (this.openValue) {
      this.show()
    }
  }

  disconnect() {
    this.dialogTarget.removeEventListener("click", this.boundHandleBackdropClick)
  }

  show() {
    if (this.modalValue) {
      this.dialogTarget.showModal()
    } else {
      this.dialogTarget.show()
    }
    this.openValue = true
  }

  close() {
    this.dialogTarget.close()
    this.openValue = false
  }

  handleBackdropClick(event) {
    // The dialog element itself is the backdrop when clicked outside content
    if (event.target === this.dialogTarget) {
      this.close()
    }
  }

  // Handle Escape key via native dialog behavior (no JS needed)
  // The <dialog> element closes on Escape by default when opened with showModal()
}
```

**HTML usage:**

```html
<div data-controller="dialog">
  <button data-action="dialog#show">Open Dialog</button>

  <dialog data-dialog-target="dialog">
    <div class="dialog-content">
      <h2>Confirm Delete</h2>
      <p>Are you sure you want to delete this item?</p>

      <div class="dialog-actions">
        <button data-action="dialog#close">Cancel</button>
        <%= button_to "Delete", item_path(@item), method: :delete,
            data: { action: "click->dialog#close" } %>
      </div>
    </div>
  </dialog>
</div>

<%# Auto-open dialog (e.g., terms of service) %>
<div data-controller="dialog" data-dialog-open-value="true">
  <dialog data-dialog-target="dialog">
    <div class="dialog-content">
      <h2>Terms of Service Updated</h2>
      <p>Please review the updated terms.</p>
      <button data-action="dialog#close">I Agree</button>
    </div>
  </dialog>
</div>
```

**How it works:** The controller wraps the native `<dialog>` element API. `showModal()` provides built-in focus trapping and Escape key handling. Backdrop clicks are detected by checking if the click target is the dialog element itself (not its children). The `modal` value controls whether to use `showModal()` (with backdrop) or `show()` (inline).

---

### 6. Local-Time Controller (40 lines)

Displays timestamps in the user's local timezone with relative formatting. Converts UTC server-rendered timestamps to human-friendly local times.

```javascript
// app/javascript/controllers/local_time_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    datetime: String,
    format: { type: String, default: "relative" },
    refreshInterval: { type: Number, default: 60000 }
  }

  connect() {
    this.update()

    if (this.formatValue === "relative") {
      this.interval = setInterval(() => this.update(), this.refreshIntervalValue)
    }
  }

  disconnect() {
    if (this.interval) {
      clearInterval(this.interval)
    }
  }

  update() {
    const date = new Date(this.datetimeValue)

    if (this.formatValue === "relative") {
      this.element.textContent = this.relativeTime(date)
    } else {
      this.element.textContent = date.toLocaleString(undefined, this.formatOptions())
    }

    this.element.title = date.toLocaleString()
  }

  relativeTime(date) {
    const seconds = Math.floor((new Date() - date) / 1000)

    if (seconds < 60) return "just now"
    if (seconds < 3600) return `${Math.floor(seconds / 60)}m ago`
    if (seconds < 86400) return `${Math.floor(seconds / 3600)}h ago`
    if (seconds < 604800) return `${Math.floor(seconds / 86400)}d ago`
    return date.toLocaleDateString()
  }

  formatOptions() {
    const formats = {
      short: { month: "short", day: "numeric", hour: "numeric", minute: "2-digit" },
      long: { weekday: "long", year: "numeric", month: "long", day: "numeric", hour: "numeric", minute: "2-digit" },
      date: { year: "numeric", month: "short", day: "numeric" },
      time: { hour: "numeric", minute: "2-digit" }
    }
    return formats[this.formatValue] || formats.short
  }
}
```

**HTML usage:**

```erb
<%# Relative time (auto-refreshing) %>
<span data-controller="local-time"
      data-local-time-datetime-value="<%= post.created_at.iso8601 %>">
  <%= post.created_at.strftime("%B %d, %Y") %>
</span>

<%# Short format (no refresh needed) %>
<span data-controller="local-time"
      data-local-time-datetime-value="<%= event.starts_at.iso8601 %>"
      data-local-time-format-value="short">
  <%= event.starts_at.strftime("%B %d, %Y %H:%M UTC") %>
</span>

<%# Long format %>
<span data-controller="local-time"
      data-local-time-datetime-value="<%= article.published_at.iso8601 %>"
      data-local-time-format-value="long">
  <%= article.published_at.strftime("%B %d, %Y") %>
</span>
```

**How it works:** The server renders a UTC timestamp in a `data-local-time-datetime-value` attribute and a fallback human-readable string as text content. When the controller connects, it replaces the text with a localized version. Relative times auto-refresh on a configurable interval. The `title` attribute always shows the full localized datetime on hover.

## Pattern Card

### GOOD: Small, focused, production-ready

All six controllers follow the same principles:

```
Clipboard:     25 lines - feature detection, async API, visual feedback
Auto-Click:     7 lines - single responsibility taken to its logical minimum
Toggle-Class:  31 lines - value-driven, CSS Classes API, reusable
Auto-Submit:   28 lines - debouncing, requestSubmit, cleanup in disconnect
Dialog:        45 lines - native <dialog>, backdrop clicks, modal/non-modal
Local-Time:    40 lines - timezone conversion, relative formatting, cleanup
```

Each controller does one thing well, uses Stimulus APIs idiomatically, and requires zero configuration beyond HTML attributes.

### BAD: Monolithic controller trying to do everything

```javascript
// DO NOT DO THIS - 200+ lines handling clipboard, dialogs, tooltips, and forms
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  connect() {
    this.initClipboard()
    this.initDialogs()
    this.initTooltips()
    this.initFormValidation()
    this.initAutoSave()
    // ... 15 more init methods
  }

  // 200+ lines of unrelated functionality
}
```

A monolithic controller is untestable, unreusable, and violates the single responsibility principle. Split it into focused controllers that compose through outlets, events, or shared HTML structure.
