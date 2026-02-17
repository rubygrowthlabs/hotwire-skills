---
title: "Strada Bridge Components"
categories:
  - hotwire-native
  - strada
  - bridge
tags:
  - strada
  - bridge-component
  - web-to-native
  - javascript
  - swift
  - kotlin
description: >-
  Two-way communication between web JavaScript and native Swift/Kotlin with Strada
  bridge components: JavaScript BridgeComponent class, iOS and Android native handlers,
  message format and lifecycle, platform detection, and component registration patterns.
---

# Strada Bridge Components

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
  - [Message Flow](#message-flow)
  - [Message Format](#message-format)
- [Implementation](#implementation)
  - [JavaScript Side: BridgeComponent Base Class](#javascript-side-bridgecomponent-base-class)
  - [Registering JavaScript Components](#registering-javascript-components)
  - [iOS Side: BridgeComponent Protocol](#ios-side-bridgecomponent-protocol)
  - [Registering iOS Components](#registering-ios-components)
  - [Android Side: BridgeComponent Class](#android-side-bridgecomponent-class)
  - [Registering Android Components](#registering-android-components)
  - [Platform Detection](#platform-detection)
  - [Lifecycle Management](#lifecycle-management)
- [Pattern Card](#pattern-card)

## Overview

Strada provides a bridge for two-way communication between your web application's JavaScript and the native Swift (iOS) or Kotlin (Android) code. It enables web pages to request native UI elements (share sheets, native menus, haptic feedback) and native code to send data back to the web page.

Bridge components follow the principle of progressive enhancement: the web page works fully without the native shell, but when running inside a Hotwire Native app, bridge components enhance the experience with platform-native UI.

Each bridge component has three parts:
1. **JavaScript component**: Runs in the web view. Sends messages to native and receives replies.
2. **iOS component**: Swift class that handles messages and presents native UIKit/SwiftUI UI.
3. **Android component**: Kotlin class that handles messages and presents native Android UI.

The JavaScript component is always the initiator. It declares what it needs (e.g., "show a share sheet with this URL"), and the native side fulfills the request using platform APIs.

## Architecture

### Message Flow

```
Web Page (JavaScript)                    Native App (Swift/Kotlin)
─────────────────────                    ────────────────────────

BridgeComponent                          BridgeComponent
  │                                        │
  ├── send("connect", data) ──────────►  onReceive(message)
  │                                        │
  │                                        ├── Present native UI
  │                                        │
  │  ◄──────────────── reply(message) ──── │
  │                                        │
  ├── onReceive(message)                   │
  │                                        │
```

1. The web page loads and the JavaScript bridge component sends a "connect" message with initial data.
2. The native component receives the message and configures its UI (e.g., adds a native bar button).
3. When the user interacts with the native UI, the native component sends a reply message back to JavaScript.
4. The JavaScript component receives the reply and performs the appropriate web action (e.g., submits a form).

### Message Format

Messages are JSON objects with a consistent structure:

```javascript
{
  "component": "form-submit",   // Component name (matches registration)
  "event": "connect",           // Event name: "connect", "submit", "disconnect", or custom
  "data": {                     // Arbitrary payload
    "title": "Save Changes",
    "submitButtonTitle": "Save"
  }
}
```

Standard events:
- **`connect`**: Sent by JavaScript when the component initializes on the page.
- **`disconnect`**: Sent by JavaScript when the component is removed from the page.
- Custom events (e.g., `submit`, `share`, `menuItemSelected`) are defined per component.

## Implementation

### JavaScript Side: BridgeComponent Base Class

Create JavaScript bridge components by extending the `BridgeComponent` class from `@hotwired/strada`:

```javascript
// app/javascript/bridge/form_submit_component.js
import { BridgeComponent } from "@hotwired/strada"

export default class extends BridgeComponent {
  static component = "form-submit"
  static targets = ["submit"]

  connect() {
    super.connect()

    // Send initial data to native when the component connects
    const submitTitle = this.submitTarget.value || "Submit"
    this.send("connect", { submitTitle }, () => {
      // Callback fired when native acknowledges the message
    })
  }

  // Called when native sends a message back (e.g., user tapped native button)
  onReceive(message) {
    if (message.event === "submit") {
      this.submitTarget.click()
    }
  }
}
```

The `BridgeComponent` class extends Stimulus `Controller`, so it has the same lifecycle hooks (`connect`, `disconnect`), targets, values, and actions.

### Registering JavaScript Components

Register all bridge components with the Strada bridge in your application entry point:

```javascript
// app/javascript/application.js
import { Application } from "@hotwired/stimulus"
import { Bridge } from "@hotwired/strada"
import FormSubmitComponent from "./bridge/form_submit_component"
import MenuComponent from "./bridge/menu_component"
import ShareComponent from "./bridge/share_component"
import AlertComponent from "./bridge/alert_component"

// Register Stimulus application
const application = Application.start()

// Register bridge components with Strada
Bridge.register(FormSubmitComponent)
Bridge.register(MenuComponent)
Bridge.register(ShareComponent)
Bridge.register(AlertComponent)
```

In your HTML, use bridge components like Stimulus controllers with the `bridge` prefix:

```erb
<%# app/views/posts/_form.html.erb %>
<%= form_with(model: post, data: { controller: "bridge--form-submit" }) do |f| %>
  <div>
    <%= f.label :title %>
    <%= f.text_field :title %>
  </div>

  <div>
    <%= f.label :body %>
    <%= f.text_area :body %>
  </div>

  <%# This button is hidden when running in native app (native bar button replaces it) %>
  <div data-bridge--form-submit-target="submit"
       class="<%= 'hidden' if turbo_native_app? %>">
    <%= f.submit "Save Post" %>
  </div>
<% end %>
```

### iOS Side: BridgeComponent Protocol

Create a native bridge component in Swift that handles messages from JavaScript:

```swift
// BridgeComponents/FormSubmitComponent.swift
import HotwireNative
import UIKit

final class FormSubmitComponent: BridgeComponent {
    override class var name: String { "form-submit" }

    private var submitTitle: String = "Submit"

    override func onReceive(message: Message) {
        guard let event = message.event else { return }

        switch event {
        case "connect":
            handleConnect(message: message)
        default:
            break
        }
    }

    private func handleConnect(message: Message) {
        guard let data = message.data else { return }

        // Read the submit button title from the JavaScript message
        submitTitle = data["submitTitle"] as? String ?? "Submit"

        // Add a native bar button item to the navigation bar
        let button = UIBarButtonItem(
            title: submitTitle,
            style: .done,
            target: self,
            action: #selector(submitTapped)
        )
        viewController?.navigationItem.rightBarButtonItem = button
    }

    @objc private func submitTapped() {
        // Send a reply back to JavaScript to trigger form submission
        reply(to: "connect", with: MessageData(event: "submit"))
    }
}
```

### Registering iOS Components

Register native bridge components with the Hotwire configuration:

```swift
// AppDelegate.swift or SceneDelegate.swift
import HotwireNative

func configureBridgeComponents() {
    Hotwire.config.bridgeComponentTypes = [
        FormSubmitComponent.self,
        MenuComponent.self,
        ShareComponent.self,
        AlertComponent.self
    ]
}
```

Call this before any web view loads, typically in `application(_:didFinishLaunchingWithOptions:)` or at the beginning of `scene(_:willConnectTo:options:)`.

### Android Side: BridgeComponent Class

Create the equivalent Kotlin bridge component:

```kotlin
// bridge/FormSubmitComponent.kt
package com.example.myapp.bridge

import android.view.Menu
import android.view.MenuItem
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.Message
import kotlinx.serialization.Serializable

class FormSubmitComponent(
    name: String,
    private val delegate: BridgeDelegate
) : BridgeComponent(name, delegate) {

    private var submitTitle: String = "Submit"

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> handleConnect(message)
        }
    }

    private fun handleConnect(message: Message) {
        val data = message.data<ConnectData>() ?: return
        submitTitle = data.submitTitle

        // Add a native menu item to the toolbar
        delegate.fragment.toolbarForNavigation()?.let { toolbar ->
            toolbar.menu.clear()
            toolbar.menu.add(Menu.NONE, SUBMIT_ITEM_ID, Menu.NONE, submitTitle).apply {
                setShowAsAction(MenuItem.SHOW_AS_ACTION_ALWAYS)
            }
            toolbar.setOnMenuItemClickListener { item ->
                if (item.itemId == SUBMIT_ITEM_ID) {
                    submitTapped()
                    true
                } else {
                    false
                }
            }
        }
    }

    private fun submitTapped() {
        // Send reply back to JavaScript to trigger form submission
        replyTo("connect", MessageData(event = "submit"))
    }

    @Serializable
    data class ConnectData(val submitTitle: String = "Submit")

    companion object {
        const val SUBMIT_ITEM_ID = 1001
    }
}
```

### Registering Android Components

Register bridge components in the Hotwire configuration:

```kotlin
// MyApplication.kt
import dev.hotwire.core.bridge.Bridge

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        Bridge.registerComponents(
            FormSubmitComponent::class,
            MenuComponent::class,
            ShareComponent::class,
            AlertComponent::class
        )
    }
}
```

### Platform Detection

Detect whether the page is running inside a Hotwire Native app to conditionally enhance the UI:

**Server-side (Rails):**

```ruby
# Available in controllers and views
turbo_native_app?  # Returns true when user agent contains "Turbo Native"
```

```erb
<%# Conditionally hide web UI that native replaces %>
<div class="<%= 'hidden' if turbo_native_app? %>">
  <button>Share</button>  <%# Web fallback, hidden when native share sheet is available %>
</div>
```

**Client-side (JavaScript):**

```javascript
// Check if Strada bridge is available
const isNative = window.Strada !== undefined

// Or check user agent
const isTurboNative = navigator.userAgent.includes("Turbo Native")
```

**In bridge components:**

Bridge components only run when the Strada bridge is present, so you do not need to check. The `connect()` method is only called when the native app is hosting the web view.

### Lifecycle Management

Bridge components follow the Stimulus lifecycle, enhanced with Strada-specific hooks:

```javascript
import { BridgeComponent } from "@hotwired/strada"

export default class extends BridgeComponent {
  static component = "my-component"

  connect() {
    super.connect()  // IMPORTANT: Always call super.connect()
    // Component is now active in the web view
    // Send initial data to native
    this.send("connect", { /* initial data */ })
  }

  disconnect() {
    super.disconnect()  // IMPORTANT: Always call super.disconnect()
    // Component is being removed from the web view
    // Native side will receive a "disconnect" event automatically
  }

  onReceive(message) {
    // Handle messages from native
    // This is called on the JavaScript side when native sends a reply
  }
}
```

On the native side, the component lifecycle mirrors the web page:

```swift
// iOS
final class MyComponent: BridgeComponent {
    override class var name: String { "my-component" }

    override func onReceive(message: Message) {
        // Called when JavaScript sends a message
    }

    // Called when the web view navigates away from the page
    // Clean up any native UI here
    override func onWebViewDidDisappear() {
        viewController?.navigationItem.rightBarButtonItem = nil
    }
}
```

## Pattern Card

### GOOD: Bridge Component With Proper Message Passing

```javascript
// JavaScript: Declares what it needs, sends structured data
import { BridgeComponent } from "@hotwired/strada"

export default class extends BridgeComponent {
  static component = "share"

  connect() {
    super.connect()
    this.send("connect", {
      url: this.element.dataset.shareUrl,
      title: this.element.dataset.shareTitle
    })
  }

  onReceive(message) {
    if (message.event === "shared") {
      // Native share completed, update UI
      this.element.textContent = "Shared!"
    }
  }
}
```

```swift
// iOS: Handles the native presentation
final class ShareComponent: BridgeComponent {
    override class var name: String { "share" }

    override func onReceive(message: Message) {
        guard message.event == "connect",
              let data = message.data,
              let urlString = data["url"] as? String,
              let url = URL(string: urlString) else { return }

        let title = data["title"] as? String ?? ""

        let button = UIBarButtonItem(
            image: UIImage(systemName: "square.and.arrow.up"),
            style: .plain,
            target: self,
            action: #selector(shareTapped)
        )
        viewController?.navigationItem.rightBarButtonItem = button

        self.shareURL = url
        self.shareTitle = title
    }

    @objc private func shareTapped() {
        let activityVC = UIActivityViewController(
            activityItems: [shareTitle, shareURL as Any],
            applicationActivities: nil
        )
        viewController?.present(activityVC, animated: true) {
            self.reply(to: "connect", with: MessageData(event: "shared"))
        }
    }
}
```

This approach keeps a clean separation: JavaScript declares data, native presents platform UI, and replies flow back through the bridge. The web page works without the native shell (the share button remains visible and functional as a web link).

### BAD: JavaScript Interface Injection Without Bridge Components

```swift
// BAD: Injecting JavaScript interfaces directly into WKWebView
class ViewController: UIViewController, WKScriptMessageHandler {
    func setupWebView() {
        let config = WKWebViewConfiguration()

        // Injecting raw message handlers bypasses the Strada lifecycle
        config.userContentController.add(self, name: "shareHandler")
        config.userContentController.add(self, name: "menuHandler")
        config.userContentController.add(self, name: "cameraHandler")

        let webView = WKWebView(frame: .zero, configuration: config)
    }

    func userContentController(
        _ controller: WKUserContentController,
        didReceive message: WKScriptMessage
    ) {
        // Raw message handling with no structure, no lifecycle management
        if message.name == "shareHandler" {
            let body = message.body as? [String: Any]
            // No reply mechanism -- one-way communication only
            // No disconnect handling -- native UI persists after page navigation
            // No component registration -- no way to know what the page supports
        }
    }
}
```

```javascript
// BAD: Calling webkit message handlers directly
document.querySelector("#share-btn").addEventListener("click", () => {
    // No bridge abstraction -- tightly coupled to WKWebView
    // Breaks on Android (no webkit.messageHandlers)
    // No lifecycle management -- no connect/disconnect
    // No structured message format
    window.webkit.messageHandlers.shareHandler.postMessage({
        url: window.location.href
    })
})
```

This approach bypasses Strada entirely: no structured message format, no lifecycle management, no cross-platform support, no reply mechanism from native to web, and native UI persists after page navigation because there is no disconnect handling. Each platform requires completely different JavaScript code.
