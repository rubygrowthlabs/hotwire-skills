---
title: Custom Turbo Stream Actions
date: 2025-01-20
categories:
  - Turbo Streams
tags:
  - turbo-streams
  - custom-actions
  - javascript
  - rails-helpers
  - StreamActions
description: Extend Turbo with custom actions beyond the 7 built-in ones using StreamActions and server-side helpers.
---

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Registering a Custom Action in JavaScript](#registering-a-custom-action-in-javascript)
  - [Server-Side Rails Helpers](#server-side-rails-helpers)
  - [Accessing Template Content](#accessing-template-content)
  - [Common Custom Actions](#common-custom-actions)
  - [Custom Actions with Attributes](#custom-actions-with-attributes)
- [Key Points](#key-points)
- [Pattern Card: Custom Stream Actions](#pattern-card-custom-stream-actions)

## Overview

Turbo provides 7 built-in stream actions (append, prepend, replace, update, remove, before, after). When these are insufficient for your UI needs, you can define custom actions. Custom actions let you perform arbitrary DOM operations, dispatch events, manage cookies, trigger redirects, or orchestrate animations -- all driven by the server via stream responses or broadcasts.

Custom actions are registered on `Turbo.StreamActions` in JavaScript. On the server side, you use `turbo_stream.action(:action_name, ...)` to emit the corresponding `<turbo-stream>` element.

## Implementation

### Registering a Custom Action in JavaScript

Register custom actions by adding methods to `Turbo.StreamActions`. Inside the method, `this` refers to the `<turbo-stream>` element:

```javascript
// app/javascript/custom_stream_actions.js
import { StreamActions } from "@hotwired/turbo"

// <turbo-stream action="console_log" message="Hello from the server"></turbo-stream>
StreamActions.console_log = function () {
  const message = this.getAttribute("message")
  console.log(`[TurboStream] ${message}`)
}
```

Import this file in your application entry point:

```javascript
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "./custom_stream_actions"
```

### Server-Side Rails Helpers

Use `turbo_stream.action` to render custom actions from ERB templates:

```erb
<%# app/views/sessions/create.turbo_stream.erb %>
<%= turbo_stream.action :console_log, "", message: "User signed in" %>
```

For frequently used custom actions, define a helper to keep templates clean:

```ruby
# app/helpers/turbo_stream_actions_helper.rb
module TurboStreamActionsHelper
  def turbo_stream_console_log(message)
    turbo_stream.action(:console_log, "", message: message)
  end

  def turbo_stream_redirect(url)
    turbo_stream.action(:redirect, "", url: url)
  end

  def turbo_stream_dispatch_event(event_name, target: nil, detail: "{}")
    turbo_stream.action(:dispatch_event, target || "", event_name: event_name, detail: detail)
  end
end
```

Alternatively, extend `Turbo::Streams::TagBuilder` for a more integrated approach:

```ruby
# config/initializers/turbo_stream_actions.rb
module CustomTurboStreamActions
  def redirect(url)
    action(:redirect, "", url: url)
  end

  def dispatch_event(target, event_name:, detail: "{}")
    action(:dispatch_event, target, event_name: event_name, detail: detail)
  end

  def set_cookie(name, value, expires: nil)
    action(:set_cookie, "", name: name, value: value, expires: expires&.iso8601 || "")
  end
end

Turbo::Streams::TagBuilder.prepend(CustomTurboStreamActions)
```

Now you can use these like built-in actions:

```erb
<%= turbo_stream.redirect edit_post_path(@post) %>
<%= turbo_stream.dispatch_event "notifications", event_name: "new-message", detail: { count: 3 }.to_json %>
<%= turbo_stream.set_cookie "theme", "dark", expires: 1.year.from_now %>
```

### Accessing Template Content

Custom actions can include HTML content in their `<template>` child, just like built-in actions. Access it via `this.templateContent`:

```javascript
// A custom action that inserts content into a modal body
StreamActions.fill_modal = function () {
  const target = document.getElementById(this.getAttribute("target"))
  if (target) {
    const body = target.querySelector(".modal-body")
    body.innerHTML = ""
    body.appendChild(this.templateContent.cloneNode(true))
    target.showModal()
  }
}
```

```erb
<%= turbo_stream.action :fill_modal, "confirmation_modal" do %>
  <p>Are you sure you want to delete "<%= @item.name %>"?</p>
  <%= button_to "Confirm", item_path(@item), method: :delete, class: "btn btn-danger" %>
<% end %>
```

### Common Custom Actions

**Redirect** -- navigate the browser to a new URL:

```javascript
StreamActions.redirect = function () {
  const url = this.getAttribute("url")
  Turbo.visit(url)
}
```

**Dispatch Event** -- fire a custom DOM event that Stimulus controllers can listen for:

```javascript
StreamActions.dispatch_event = function () {
  const eventName = this.getAttribute("event_name")
  const detail = JSON.parse(this.getAttribute("detail") || "{}")
  const target = this.getAttribute("target")

  const element = target ? document.getElementById(target) : document
  element?.dispatchEvent(new CustomEvent(eventName, { detail, bubbles: true }))
}
```

**Set Cookie** -- store a value in a browser cookie:

```javascript
StreamActions.set_cookie = function () {
  const name = this.getAttribute("name")
  const value = this.getAttribute("value")
  const expires = this.getAttribute("expires")

  let cookie = `${name}=${encodeURIComponent(value)}; path=/; SameSite=Lax`
  if (expires) cookie += `; expires=${expires}`
  document.cookie = cookie
}
```

**Scroll To** -- scroll to a specific element:

```javascript
StreamActions.scroll_to = function () {
  const target = document.getElementById(this.getAttribute("target"))
  target?.scrollIntoView({ behavior: "smooth", block: "start" })
}
```

### Custom Actions with Attributes

Custom actions can read arbitrary attributes from the `<turbo-stream>` element. Pass them from Rails using keyword arguments to `turbo_stream.action`:

```erb
<%= turbo_stream.action :notification, "",
      level: "success",
      message: "Item saved successfully",
      duration: "3000" %>
```

```javascript
StreamActions.notification = function () {
  const level = this.getAttribute("level") || "info"
  const message = this.getAttribute("message")
  const duration = parseInt(this.getAttribute("duration") || "5000", 10)

  const container = document.getElementById("notifications")
  const el = document.createElement("div")
  el.className = `notification notification--${level}`
  el.textContent = message
  container?.appendChild(el)

  setTimeout(() => el.remove(), duration)
}
```

## Key Points

- Register custom actions on `Turbo.StreamActions` in JavaScript
- Inside a custom action, `this` refers to the `<turbo-stream>` element
- Use `this.templateContent` to access the `<template>` child's content
- Use `turbo_stream.action(:name, target, **attributes)` on the server side
- Extend `Turbo::Streams::TagBuilder` for clean, reusable server-side helpers
- Custom actions run for both HTTP responses and WebSocket broadcasts
- Keep custom actions focused on a single responsibility

## Pattern Card: Custom Stream Actions

**When to use**: UI behavior that cannot be expressed with the 7 built-in stream actions (append, prepend, replace, update, remove, before, after).

**GOOD - Focused custom action with clear responsibility**:

```javascript
// app/javascript/custom_stream_actions.js
import { StreamActions } from "@hotwired/turbo"

StreamActions.redirect = function () {
  Turbo.visit(this.getAttribute("url"))
}

StreamActions.dispatch_event = function () {
  const name = this.getAttribute("event_name")
  const detail = JSON.parse(this.getAttribute("detail") || "{}")
  const target = this.getAttribute("target")
  const el = target ? document.getElementById(target) : document
  el?.dispatchEvent(new CustomEvent(name, { detail, bubbles: true }))
}
```

```ruby
# config/initializers/turbo_stream_actions.rb
module CustomTurboStreamActions
  def redirect(url)
    action(:redirect, "", url: url)
  end

  def dispatch_event(target, event_name:, detail: "{}")
    action(:dispatch_event, target, event_name: event_name, detail: detail)
  end
end

Turbo::Streams::TagBuilder.prepend(CustomTurboStreamActions)
```

```erb
<%# Clean usage in templates %>
<%= turbo_stream.redirect root_path %>
<%= turbo_stream.dispatch_event "inbox", event_name: "refresh", detail: { unread: 5 }.to_json %>
```

**BAD - Overloading built-in actions or embedding scripts**:

```erb
<%# Don't override built-in action behavior %>
<%= turbo_stream.replace "page" do %>
  <script>window.location.href = '<%= root_path %>'</script>
<% end %>

<%# Don't put unrelated logic in a replace action %>
<%= turbo_stream.replace "dummy" do %>
  <div id="dummy" style="display:none"
       data-controller="side-effect"
       data-side-effect-url-value="<%= root_path %>">
  </div>
<% end %>
```
