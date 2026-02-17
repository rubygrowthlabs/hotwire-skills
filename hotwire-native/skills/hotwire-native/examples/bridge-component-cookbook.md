---
title: "Bridge Component Cookbook"
categories:
  - hotwire-native
  - bridge
  - examples
tags:
  - bridge-component
  - share-sheet
  - native-menu
  - form-submit
  - native-alert
  - cookbook
description: >-
  Four complete bridge component implementations with full JavaScript, Swift, and HTML
  code: Share Sheet (native UIActivityViewController), Native Menu (UIMenu on navigation
  bar), Form Submit Button (native bar button triggers web form), and Native Alert
  (UIAlertController from web trigger).
---

# Bridge Component Cookbook

## Table of Contents

- [Overview](#overview)
- [1. Share Sheet](#1-share-sheet)
  - [JavaScript: Share Bridge Component](#javascript-share-bridge-component)
  - [Swift: Native Share Handler](#swift-native-share-handler)
  - [Kotlin: Native Share Handler](#kotlin-native-share-handler)
  - [HTML: Share Markup](#html-share-markup)
- [2. Native Menu](#2-native-menu)
  - [JavaScript: Menu Bridge Component](#javascript-menu-bridge-component)
  - [Swift: Native Menu Handler](#swift-native-menu-handler)
  - [Kotlin: Native Menu Handler](#kotlin-native-menu-handler)
  - [HTML: Menu Markup](#html-menu-markup)
- [3. Form Submit Button](#3-form-submit-button)
  - [JavaScript: Form Submit Bridge Component](#javascript-form-submit-bridge-component)
  - [Swift: Native Submit Handler](#swift-native-submit-handler)
  - [Kotlin: Native Submit Handler](#kotlin-native-submit-handler)
  - [HTML: Form Markup](#html-form-markup)
- [4. Native Alert](#4-native-alert)
  - [JavaScript: Alert Bridge Component](#javascript-alert-bridge-component)
  - [Swift: Native Alert Handler](#swift-native-alert-handler)
  - [Kotlin: Native Alert Handler](#kotlin-native-alert-handler)
  - [HTML: Alert Markup](#html-alert-markup)
- [Registration Summary](#registration-summary)

## Overview

This cookbook provides four complete bridge component implementations. Each component follows the same pattern:

1. **JavaScript bridge component** (extends `BridgeComponent` from `@hotwired/hotwire-native-bridge`): Runs in the web view, sends data to native, receives replies.
2. **Swift native handler** (extends `BridgeComponent` from `HotwireNative`): Presents iOS-native UI, sends replies back to JavaScript.
3. **Kotlin native handler** (extends `BridgeComponent` from `dev.hotwire.core`): Presents Android-native UI, sends replies back to JavaScript.
4. **HTML markup**: The ERB template with data attributes that wire everything together.

Every component follows the progressive enhancement principle: the web version works without the native shell. When running inside a Hotwire Native app, the native UI replaces or augments the web UI.

---

## 1. Share Sheet

**Purpose**: A web page has content the user can share. In a browser, a standard share link is shown. In the native app, tapping the share button opens the platform's native share sheet (UIActivityViewController on iOS, Intent.ACTION_SEND on Android).

### JavaScript: Share Bridge Component

```javascript
// app/javascript/bridge/share_component.js
import { BridgeComponent, BridgeElement } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "share"
  static targets = ["button"]

  connect() {
    super.connect()

    // Send share data to native on connect
    const shareData = {
      url: this.element.dataset.shareUrl,
      title: this.element.dataset.shareTitle,
      text: this.element.dataset.shareText || ""
    }

    this.send("connect", shareData, () => {
      // Native acknowledged -- hide the web share button
      if (this.hasButtonTarget) {
        this.buttonTarget.hidden = true
      }
    })
  }

  onReceive(message) {
    if (message.event === "shared") {
      // User completed sharing via native sheet
      // Optionally update UI to show "Shared!" confirmation
      if (this.hasButtonTarget) {
        this.buttonTarget.textContent = "Shared!"
        setTimeout(() => {
          this.buttonTarget.textContent = "Share"
        }, 2000)
      }
    }
  }

  // Web fallback: use Web Share API if available, otherwise copy to clipboard
  webShare(event) {
    event.preventDefault()
    const shareData = {
      url: this.element.dataset.shareUrl,
      title: this.element.dataset.shareTitle,
      text: this.element.dataset.shareText || ""
    }

    if (navigator.share) {
      navigator.share(shareData)
    } else {
      navigator.clipboard.writeText(shareData.url)
      this.buttonTarget.textContent = "Link copied!"
      setTimeout(() => {
        this.buttonTarget.textContent = "Share"
      }, 2000)
    }
  }
}
```

### Swift: Native Share Handler

```swift
// BridgeComponents/ShareComponent.swift
import HotwireNative
import UIKit

final class ShareComponent: BridgeComponent {
    override class var name: String { "share" }

    private var shareURL: URL?
    private var shareTitle: String = ""
    private var shareText: String = ""

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

        shareURL = (data["url"] as? String).flatMap { URL(string: $0) }
        shareTitle = data["title"] as? String ?? ""
        shareText = data["text"] as? String ?? ""

        // Add a share button to the navigation bar
        let shareButton = UIBarButtonItem(
            image: UIImage(systemName: "square.and.arrow.up"),
            style: .plain,
            target: self,
            action: #selector(shareTapped)
        )
        viewController?.navigationItem.rightBarButtonItem = shareButton
    }

    @objc private func shareTapped() {
        var items: [Any] = []

        if !shareTitle.isEmpty { items.append(shareTitle) }
        if !shareText.isEmpty { items.append(shareText) }
        if let url = shareURL { items.append(url) }

        guard !items.isEmpty else { return }

        let activityVC = UIActivityViewController(
            activityItems: items,
            applicationActivities: nil
        )

        // Required for iPad -- set popover source
        if let popover = activityVC.popoverPresentationController {
            popover.barButtonItem = viewController?.navigationItem.rightBarButtonItem
        }

        viewController?.present(activityVC, animated: true) { [weak self] in
            // Notify JavaScript that sharing completed
            self?.reply(to: "connect", with: MessageData(event: "shared"))
        }
    }

    override func onWebViewDidDisappear() {
        // Clean up navigation bar when leaving the page
        viewController?.navigationItem.rightBarButtonItem = nil
    }
}
```

### Kotlin: Native Share Handler

```kotlin
// bridge/ShareComponent.kt
package com.example.myapp.bridge

import android.content.Intent
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.Message
import kotlinx.serialization.Serializable

class ShareComponent(
    name: String,
    private val delegate: BridgeDelegate
) : BridgeComponent(name, delegate) {

    private var shareUrl: String = ""
    private var shareTitle: String = ""
    private var shareText: String = ""

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> handleConnect(message)
        }
    }

    private fun handleConnect(message: Message) {
        val data = message.data<ShareData>() ?: return
        shareUrl = data.url
        shareTitle = data.title
        shareText = data.text

        // Add share action to toolbar
        delegate.fragment.toolbarForNavigation()?.let { toolbar ->
            toolbar.menu.clear()
            toolbar.inflateMenu(R.menu.share_menu)
            toolbar.setOnMenuItemClickListener { item ->
                if (item.itemId == R.id.action_share) {
                    shareTapped()
                    true
                } else false
            }
        }
    }

    private fun shareTapped() {
        val shareIntent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_SUBJECT, shareTitle)
            putExtra(Intent.EXTRA_TEXT, "$shareText\n$shareUrl")
        }

        val chooser = Intent.createChooser(shareIntent, "Share via")
        delegate.fragment.startActivity(chooser)

        // Notify JavaScript
        replyTo("connect", MessageData(event = "shared"))
    }

    @Serializable
    data class ShareData(
        val url: String = "",
        val title: String = "",
        val text: String = ""
    )
}
```

### HTML: Share Markup

```erb
<%# app/views/posts/show.html.erb %>
<article>
  <h1><%= @post.title %></h1>
  <%= simple_format(@post.body) %>

  <%# Bridge component: native share sheet in Hotwire Native, web fallback in browser %>
  <div data-controller="bridge--share"
       data-share-url="<%= post_url(@post) %>"
       data-share-title="<%= @post.title %>"
       data-share-text="Check out this post!">

    <%# Web fallback button (hidden when native bridge is active) %>
    <button data-bridge--share-target="button"
            data-action="click->bridge--share#webShare"
            class="btn btn-secondary">
      Share
    </button>
  </div>
</article>
```

---

## 2. Native Menu

**Purpose**: A web page defines a list of actions (edit, delete, archive). In a browser, these are shown as buttons or a dropdown. In the native app, they appear as a native UIMenu (iOS) or overflow menu (Android) on the navigation bar.

### JavaScript: Menu Bridge Component

```javascript
// app/javascript/bridge/menu_component.js
import { BridgeComponent, BridgeElement } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "menu"

  connect() {
    super.connect()

    // Parse menu items from data attribute
    const items = JSON.parse(this.element.dataset.menuItems || "[]")

    this.send("connect", { items }, () => {
      // Native acknowledged -- optionally hide web dropdown
      const webMenu = this.element.querySelector("[data-menu-web]")
      if (webMenu) webMenu.hidden = true
    })
  }

  onReceive(message) {
    if (message.event === "itemSelected") {
      const selectedEvent = message.data?.event
      this.handleMenuSelection(selectedEvent)
    }
  }

  handleMenuSelection(event) {
    switch (event) {
      case "edit":
        const editUrl = this.element.dataset.menuEditUrl
        if (editUrl) Turbo.visit(editUrl)
        break
      case "delete":
        const deleteForm = this.element.querySelector("[data-menu-delete-form]")
        if (deleteForm) deleteForm.requestSubmit()
        break
      case "archive":
        const archiveForm = this.element.querySelector("[data-menu-archive-form]")
        if (archiveForm) archiveForm.requestSubmit()
        break
      case "share":
        // Delegate to share component or use Web Share API
        if (navigator.share) {
          navigator.share({ url: window.location.href })
        }
        break
    }
  }
}
```

### Swift: Native Menu Handler

```swift
// BridgeComponents/MenuComponent.swift
import HotwireNative
import UIKit

final class MenuComponent: BridgeComponent {
    override class var name: String { "menu" }

    private var menuItems: [[String: Any]] = []

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
        guard let data = message.data,
              let items = data["items"] as? [[String: Any]] else { return }

        self.menuItems = items

        // Build UIMenu from the items
        let actions = items.compactMap { item -> UIAction? in
            guard let title = item["title"] as? String,
                  let event = item["event"] as? String else { return nil }

            let icon = (item["icon"] as? String).flatMap { UIImage(systemName: $0) }
            let isDestructive = item["destructive"] as? Bool ?? false

            return UIAction(
                title: title,
                image: icon,
                attributes: isDestructive ? .destructive : []
            ) { [weak self] _ in
                self?.menuItemSelected(event: event)
            }
        }

        let menu = UIMenu(title: "", children: actions)

        let menuButton = UIBarButtonItem(
            image: UIImage(systemName: "ellipsis.circle"),
            menu: menu
        )
        viewController?.navigationItem.rightBarButtonItem = menuButton
    }

    private func menuItemSelected(event: String) {
        // Send the selected event back to JavaScript
        reply(to: "connect", with: MessageData(
            event: "itemSelected",
            data: ["event": event]
        ))
    }

    override func onWebViewDidDisappear() {
        viewController?.navigationItem.rightBarButtonItem = nil
    }
}
```

### Kotlin: Native Menu Handler

```kotlin
// bridge/MenuComponent.kt
package com.example.myapp.bridge

import android.view.Menu
import android.view.MenuItem
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.Message
import kotlinx.serialization.Serializable

class MenuComponent(
    name: String,
    private val delegate: BridgeDelegate
) : BridgeComponent(name, delegate) {

    private var menuItems: List<MenuItemData> = emptyList()

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> handleConnect(message)
        }
    }

    private fun handleConnect(message: Message) {
        val data = message.data<ConnectData>() ?: return
        menuItems = data.items

        delegate.fragment.toolbarForNavigation()?.let { toolbar ->
            toolbar.menu.clear()

            menuItems.forEachIndexed { index, item ->
                toolbar.menu.add(Menu.NONE, index, Menu.NONE, item.title).apply {
                    setShowAsAction(
                        if (index == 0) MenuItem.SHOW_AS_ACTION_IF_ROOM
                        else MenuItem.SHOW_AS_ACTION_NEVER
                    )
                }
            }

            toolbar.setOnMenuItemClickListener { menuItem ->
                val selectedItem = menuItems.getOrNull(menuItem.itemId)
                if (selectedItem != null) {
                    menuItemSelected(selectedItem.event)
                    true
                } else false
            }
        }
    }

    private fun menuItemSelected(event: String) {
        replyTo("connect", MessageData(
            event = "itemSelected",
            data = mapOf("event" to event)
        ))
    }

    @Serializable
    data class ConnectData(val items: List<MenuItemData>)

    @Serializable
    data class MenuItemData(
        val title: String,
        val icon: String = "",
        val event: String,
        val destructive: Boolean = false
    )
}
```

### HTML: Menu Markup

```erb
<%# app/views/posts/show.html.erb %>
<article>
  <h1><%= @post.title %></h1>
  <%= simple_format(@post.body) %>

  <%# Bridge component: native overflow menu in Hotwire Native, web dropdown in browser %>
  <div data-controller="bridge--menu"
       data-menu-items='<%= [
         { title: "Edit", icon: "pencil", event: "edit" },
         { title: "Share", icon: "square.and.arrow.up", event: "share" },
         { title: "Archive", icon: "archivebox", event: "archive" },
         { title: "Delete", icon: "trash", event: "delete", destructive: true }
       ].to_json %>'
       data-menu-edit-url="<%= edit_post_path(@post) %>">

    <%# Web fallback: dropdown menu (hidden when native bridge is active) %>
    <div data-menu-web class="dropdown">
      <button class="btn btn-secondary dropdown-toggle">Actions</button>
      <div class="dropdown-menu">
        <%= link_to "Edit", edit_post_path(@post), class: "dropdown-item" %>
        <button class="dropdown-item" data-action="click->share#open">Share</button>
        <%= button_to "Archive", archive_post_path(@post),
            method: :patch, class: "dropdown-item",
            form: { data: { menu_archive_form: true } } %>
        <%= button_to "Delete", @post, method: :delete,
            class: "dropdown-item text-danger",
            form: { data: { turbo_confirm: "Delete this post?", menu_delete_form: true } } %>
      </div>
    </div>
  </div>
</article>
```

---

## 3. Form Submit Button

**Purpose**: Forms in web views have a submit button at the bottom of the page. In the native app, the submit action moves to a native bar button in the navigation bar, matching the iOS/Android convention for "Save" buttons.

### JavaScript: Form Submit Bridge Component

```javascript
// app/javascript/bridge/form_submit_component.js
import { BridgeComponent, BridgeElement } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "form-submit"
  static targets = ["submit", "form"]

  connect() {
    super.connect()

    const submitTitle = this.submitTarget.value ||
                        this.submitTarget.textContent ||
                        "Submit"

    this.send("connect", { submitTitle }, () => {
      // Native acknowledged -- hide the web submit button
      this.submitTarget.hidden = true
    })
  }

  onReceive(message) {
    if (message.event === "submit") {
      // Native bar button was tapped -- submit the web form
      this.formTarget.requestSubmit()
    }
  }

  // Handle form submission state
  submitStart() {
    // Notify native to show loading state on the bar button
    this.send("submitStart", {})
  }

  submitEnd() {
    // Notify native to restore the bar button
    this.send("submitEnd", {})
  }
}
```

### Swift: Native Submit Handler

```swift
// BridgeComponents/FormSubmitComponent.swift
import HotwireNative
import UIKit

final class FormSubmitComponent: BridgeComponent {
    override class var name: String { "form-submit" }

    private var submitButton: UIBarButtonItem?
    private var loadingIndicator: UIActivityIndicatorView?

    override func onReceive(message: Message) {
        guard let event = message.event else { return }

        switch event {
        case "connect":
            handleConnect(message: message)
        case "submitStart":
            showLoadingState()
        case "submitEnd":
            hideLoadingState()
        default:
            break
        }
    }

    private func handleConnect(message: Message) {
        let submitTitle = message.data?["submitTitle"] as? String ?? "Submit"

        submitButton = UIBarButtonItem(
            title: submitTitle,
            style: .done,
            target: self,
            action: #selector(submitTapped)
        )
        submitButton?.tintColor = .systemBlue

        viewController?.navigationItem.rightBarButtonItem = submitButton
    }

    @objc private func submitTapped() {
        // Send reply to JavaScript to trigger form submission
        reply(to: "connect", with: MessageData(event: "submit"))

        // Provide haptic feedback
        let generator = UIImpactFeedbackGenerator(style: .medium)
        generator.impactOccurred()
    }

    private func showLoadingState() {
        let indicator = UIActivityIndicatorView(style: .medium)
        indicator.startAnimating()
        loadingIndicator = indicator

        viewController?.navigationItem.rightBarButtonItem = UIBarButtonItem(
            customView: indicator
        )
    }

    private func hideLoadingState() {
        viewController?.navigationItem.rightBarButtonItem = submitButton
    }

    override func onWebViewDidDisappear() {
        viewController?.navigationItem.rightBarButtonItem = nil
    }
}
```

### Kotlin: Native Submit Handler

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
            "submitStart" -> showLoadingState()
            "submitEnd" -> hideLoadingState()
        }
    }

    private fun handleConnect(message: Message) {
        val data = message.data<ConnectData>() ?: return
        submitTitle = data.submitTitle

        delegate.fragment.toolbarForNavigation()?.let { toolbar ->
            toolbar.menu.clear()
            toolbar.menu.add(Menu.NONE, SUBMIT_ITEM_ID, Menu.NONE, submitTitle).apply {
                setShowAsAction(MenuItem.SHOW_AS_ACTION_ALWAYS)
            }
            toolbar.setOnMenuItemClickListener { item ->
                if (item.itemId == SUBMIT_ITEM_ID) {
                    submitTapped()
                    true
                } else false
            }
        }
    }

    private fun submitTapped() {
        replyTo("connect", MessageData(event = "submit"))

        // Haptic feedback
        delegate.fragment.view?.performHapticFeedback(
            android.view.HapticFeedbackConstants.CONFIRM
        )
    }

    private fun showLoadingState() {
        delegate.fragment.toolbarForNavigation()?.let { toolbar ->
            toolbar.menu.findItem(SUBMIT_ITEM_ID)?.isEnabled = false
            toolbar.menu.findItem(SUBMIT_ITEM_ID)?.title = "Saving..."
        }
    }

    private fun hideLoadingState() {
        delegate.fragment.toolbarForNavigation()?.let { toolbar ->
            toolbar.menu.findItem(SUBMIT_ITEM_ID)?.isEnabled = true
            toolbar.menu.findItem(SUBMIT_ITEM_ID)?.title = submitTitle
        }
    }

    @Serializable
    data class ConnectData(val submitTitle: String = "Submit")

    companion object {
        const val SUBMIT_ITEM_ID = 2001
    }
}
```

### HTML: Form Markup

```erb
<%# app/views/posts/_form.html.erb %>
<%= form_with(
      model: post,
      data: {
        controller: "bridge--form-submit",
        action: "turbo:submit-start->bridge--form-submit#submitStart turbo:submit-end->bridge--form-submit#submitEnd"
      }
    ) do |f| %>

  <div data-bridge--form-submit-target="form">
    <div class="field">
      <%= f.label :title %>
      <%= f.text_field :title, class: "form-control" %>
    </div>

    <div class="field">
      <%= f.label :body %>
      <%= f.text_area :body, class: "form-control", rows: 10 %>
    </div>

    <div class="field">
      <%= f.label :category %>
      <%= f.select :category, Post::CATEGORIES, { prompt: "Select category" }, class: "form-control" %>
    </div>
  </div>

  <%# Web submit button (hidden when native bar button is active) %>
  <div data-bridge--form-submit-target="submit">
    <%= f.submit class: "btn btn-primary mt-3" %>
  </div>
<% end %>
```

---

## 4. Native Alert

**Purpose**: A web page needs to show a confirmation or notification dialog. In a browser, this might use a modal or inline alert. In the native app, it presents a native UIAlertController (iOS) or MaterialAlertDialog (Android).

### JavaScript: Alert Bridge Component

```javascript
// app/javascript/bridge/alert_component.js
import { BridgeComponent, BridgeElement } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "alert"

  connect() {
    super.connect()

    const alertData = {
      title: this.element.dataset.alertTitle || "Alert",
      message: this.element.dataset.alertMessage || "",
      actions: JSON.parse(this.element.dataset.alertActions || '[{"title": "OK", "event": "dismiss"}]')
    }

    this.send("connect", alertData)
  }

  onReceive(message) {
    if (message.event === "actionSelected") {
      const action = message.data?.event
      this.handleAction(action)
    }
  }

  handleAction(action) {
    switch (action) {
      case "confirm":
        // Find and submit the confirmation form
        const confirmForm = this.element.querySelector("[data-alert-confirm-form]")
        if (confirmForm) confirmForm.requestSubmit()
        break
      case "dismiss":
        // Remove the alert element
        this.element.remove()
        break
      case "cancel":
        // Do nothing, dismiss handled by native
        break
    }
  }

  // Web fallback: show a browser confirm dialog
  showWebAlert(event) {
    event.preventDefault()
    const title = this.element.dataset.alertTitle || "Confirm"
    const message = this.element.dataset.alertMessage || "Are you sure?"

    if (confirm(`${title}\n\n${message}`)) {
      this.handleAction("confirm")
    }
  }
}
```

### Swift: Native Alert Handler

```swift
// BridgeComponents/AlertComponent.swift
import HotwireNative
import UIKit

final class AlertComponent: BridgeComponent {
    override class var name: String { "alert" }

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

        let title = data["title"] as? String ?? "Alert"
        let messageText = data["message"] as? String ?? ""
        let actions = data["actions"] as? [[String: Any]] ?? [["title": "OK", "event": "dismiss"]]

        let alert = UIAlertController(
            title: title,
            message: messageText,
            preferredStyle: .alert
        )

        for action in actions {
            guard let actionTitle = action["title"] as? String,
                  let event = action["event"] as? String else { continue }

            let isDestructive = action["destructive"] as? Bool ?? false
            let isCancel = action["style"] as? String == "cancel"

            let style: UIAlertAction.Style
            if isDestructive {
                style = .destructive
            } else if isCancel {
                style = .cancel
            } else {
                style = .default
            }

            alert.addAction(UIAlertAction(title: actionTitle, style: style) { [weak self] _ in
                self?.reply(to: "connect", with: MessageData(
                    event: "actionSelected",
                    data: ["event": event]
                ))
            })
        }

        viewController?.present(alert, animated: true)
    }
}
```

### Kotlin: Native Alert Handler

```kotlin
// bridge/AlertComponent.kt
package com.example.myapp.bridge

import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.Message
import kotlinx.serialization.Serializable

class AlertComponent(
    name: String,
    private val delegate: BridgeDelegate
) : BridgeComponent(name, delegate) {

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> handleConnect(message)
        }
    }

    private fun handleConnect(message: Message) {
        val data = message.data<AlertData>() ?: return

        val builder = MaterialAlertDialogBuilder(delegate.fragment.requireContext())
            .setTitle(data.title)
            .setMessage(data.message)

        // Add action buttons (max 3 for AlertDialog: positive, negative, neutral)
        val actions = data.actions
        actions.getOrNull(0)?.let { action ->
            builder.setPositiveButton(action.title) { _, _ ->
                actionSelected(action.event)
            }
        }
        actions.getOrNull(1)?.let { action ->
            builder.setNegativeButton(action.title) { _, _ ->
                actionSelected(action.event)
            }
        }
        actions.getOrNull(2)?.let { action ->
            builder.setNeutralButton(action.title) { _, _ ->
                actionSelected(action.event)
            }
        }

        builder.show()
    }

    private fun actionSelected(event: String) {
        replyTo("connect", MessageData(
            event = "actionSelected",
            data = mapOf("event" to event)
        ))
    }

    @Serializable
    data class AlertData(
        val title: String = "Alert",
        val message: String = "",
        val actions: List<AlertAction> = listOf(AlertAction("OK", "dismiss"))
    )

    @Serializable
    data class AlertAction(
        val title: String,
        val event: String,
        val destructive: Boolean = false,
        val style: String = "default"
    )
}
```

### HTML: Alert Markup

```erb
<%# Example 1: Confirmation before destructive action %>
<div data-controller="bridge--alert"
     data-alert-title="Delete Post"
     data-alert-message="This action cannot be undone. Are you sure you want to delete this post?"
     data-alert-actions='<%= [
       { title: "Delete", event: "confirm", destructive: true },
       { title: "Cancel", event: "cancel", style: "cancel" }
     ].to_json %>'>

  <%# Hidden form for the destructive action %>
  <%= button_to "Delete", @post, method: :delete,
      form: { data: { alert_confirm_form: true }, hidden: true } %>

  <%# Web fallback button %>
  <button data-action="click->bridge--alert#showWebAlert"
          class="btn btn-danger">
    Delete Post
  </button>
</div>

<%# Example 2: Success notification after save (auto-dismiss) %>
<% if flash[:notice] %>
  <div data-controller="bridge--alert"
       data-alert-title="Success"
       data-alert-message="<%= flash[:notice] %>"
       data-alert-actions='<%= [{ title: "OK", event: "dismiss" }].to_json %>'>
  </div>
<% end %>

<%# Example 3: Unsaved changes warning %>
<div data-controller="bridge--alert"
     data-alert-title="Unsaved Changes"
     data-alert-message="You have unsaved changes. Do you want to discard them?"
     data-alert-actions='<%= [
       { title: "Discard", event: "confirm", destructive: true },
       { title: "Keep Editing", event: "cancel", style: "cancel" }
     ].to_json %>'>
</div>
```

---

## Registration Summary

Register all bridge components in each platform:

**JavaScript (shared across web and native):**

Bridge components are Stimulus controllers -- register them like any other controller.
With importmap-rails or esbuild autoloading, controllers in `app/javascript/controllers/bridge/`
register automatically via the `bridge--` prefix.

Manual registration:

```javascript
// app/javascript/application.js
import { Application } from "@hotwired/stimulus"
import ShareController from "./controllers/bridge/share_controller"
import MenuController from "./controllers/bridge/menu_controller"
import FormSubmitController from "./controllers/bridge/form_submit_controller"
import AlertController from "./controllers/bridge/alert_controller"

const application = Application.start()
application.register("bridge--share", ShareController)
application.register("bridge--menu", MenuController)
application.register("bridge--form-submit", FormSubmitController)
application.register("bridge--alert", AlertController)
```

**iOS (Swift):**

```swift
// Configure in AppDelegate or SceneDelegate
Hotwire.registerBridgeComponents([
    ShareComponent.self,
    MenuComponent.self,
    FormSubmitComponent.self,
    AlertComponent.self
])
```

**Android (Kotlin):**

```kotlin
// Configure in Application.onCreate()
Bridge.registerComponents(
    ShareComponent::class,
    MenuComponent::class,
    FormSubmitComponent::class,
    AlertComponent::class
)
```
