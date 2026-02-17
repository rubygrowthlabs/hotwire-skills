---
title: "Path Configuration"
categories:
  - hotwire-native
  - routing
  - configuration
tags:
  - path-configuration
  - server-driven
  - routing
  - json
  - navigation
description: >-
  Server-driven routing with JSON path configuration: URL pattern matching with regex,
  presentation styles (default, modal, pop, refresh, none, replace), context management,
  pull-to-refresh per path, Rails controller endpoint, and caching strategies. The
  central mechanism for controlling Hotwire Native app navigation from the server.
---

# Path Configuration

## Table of Contents

- [Overview](#overview)
- [JSON Structure](#json-structure)
- [Implementation](#implementation)
  - [URL Pattern Matching](#url-pattern-matching)
  - [Presentation Styles](#presentation-styles)
  - [Context: Default vs Modal](#context-default-vs-modal)
  - [Custom Properties](#custom-properties)
  - [Pull-to-Refresh Per Path](#pull-to-refresh-per-path)
  - [Serving From Rails](#serving-from-rails)
  - [Loading in iOS](#loading-in-ios)
  - [Loading in Android](#loading-in-android)
  - [Caching and Updating](#caching-and-updating)
- [Pattern Card](#pattern-card)

## Overview

Path configuration is the central routing mechanism in Hotwire Native. It is a JSON file served from your Rails application that tells the native client how to present each URL pattern. Instead of hardcoding navigation decisions in Swift or Kotlin, the server controls whether a URL should push, present as a modal, replace the current screen, or hand off to a native view controller.

This server-driven approach is the single most important architectural decision in Hotwire Native. It enables:

- **Instant behavior changes** without App Store or Play Store releases. Change a URL from push to modal by updating the JSON on your server.
- **Feature flags** for native screens. Roll out a native camera screen to 10% of users by updating path configuration.
- **Consistent routing** across iOS and Android from a single source of truth.
- **Graceful degradation** when paths are not matched -- the default behavior is a standard web view push.

## JSON Structure

A path configuration file has two top-level keys:

```json
{
  "settings": {
    "screenshots_enabled": true,
    "tabs": [
      { "title": "Home", "path": "/", "icon": "house" },
      { "title": "Search", "path": "/search", "icon": "magnifyingglass" },
      { "title": "Profile", "path": "/profile", "icon": "person" }
    ]
  },
  "rules": [
    {
      "patterns": ["/.*"],
      "properties": {
        "context": "default",
        "presentation": "default",
        "pull_to_refresh_enabled": true
      }
    }
  ]
}
```

- **`settings`**: Global configuration read by the native app at startup. Use this for tab definitions, feature flags, minimum app version, or any key-value data the native app needs.
- **`rules`**: An ordered array of routing rules. Each rule has URL patterns and properties. Rules are evaluated top to bottom; the last matching rule wins.

## Implementation

### URL Pattern Matching

Each rule contains a `patterns` array of regex strings. When the native app is about to navigate to a URL, it tests the URL path against each rule's patterns. The last matching rule's properties are used.

```json
{
  "rules": [
    {
      "patterns": ["/.*"],
      "properties": {
        "context": "default",
        "presentation": "default"
      }
    },
    {
      "patterns": ["/new$", "/edit$"],
      "properties": {
        "context": "modal",
        "presentation": "default"
      }
    },
    {
      "patterns": ["/settings"],
      "properties": {
        "context": "modal",
        "presentation": "default",
        "pull_to_refresh_enabled": false
      }
    }
  ]
}
```

Pattern matching rules:
- Patterns match against the URL path only (not query string or fragment).
- Standard regex syntax is supported: `^`, `$`, `.*`, character classes, etc.
- Multiple patterns in the array are OR-matched: if any pattern matches, the rule applies.
- Rules are evaluated in order; the **last** matching rule's properties are merged on top of earlier matches.

### Presentation Styles

The `presentation` property controls how the native app transitions to the new screen:

| Presentation | Behavior | Use Case |
|-------------|----------|----------|
| `default` | Push onto the navigation stack (standard drill-down) | Most pages |
| `modal` | Present as a modal sheet over the current screen | Forms, settings, detail views |
| `pop` | Pop the current screen off the navigation stack | After form submission (redirect back) |
| `refresh` | Reload the current screen in place | After successful save |
| `none` | Do nothing -- ignore this navigation | Anchor links, JavaScript-only actions |
| `replace` | Replace the current screen without animation | Tab switches, redirects |
| `replace_root` | Replace the entire navigation stack | Post-login redirect |
| `clear_all` | Clear all navigation stacks and start fresh | Logout |

```json
{
  "rules": [
    {
      "patterns": ["/.*"],
      "properties": {
        "presentation": "default"
      }
    },
    {
      "patterns": ["/new$", "/edit$", "/login"],
      "properties": {
        "presentation": "default",
        "context": "modal"
      }
    },
    {
      "patterns": ["/logout"],
      "properties": {
        "presentation": "clear_all"
      }
    }
  ]
}
```

### Context: Default vs Modal

The `context` property determines which navigation stack a screen belongs to:

- **`default`**: The screen is pushed onto the main navigation stack.
- **`modal`**: The screen is presented in a modal navigation stack on top of the main stack.

```json
{
  "patterns": ["/posts/new", "/posts/.*/edit"],
  "properties": {
    "context": "modal",
    "presentation": "default"
  }
}
```

When `context` is `modal`, the native app creates a new modal navigation controller (iOS) or dialog fragment (Android). Subsequent navigations within the modal push onto the modal's stack. When the modal is dismissed, the user returns to the main stack.

This is ideal for multi-step flows (new post -> preview -> confirm) that should not pollute the main navigation history.

### Custom Properties

You can add any custom key-value pairs to properties. The native app reads these to make additional decisions:

```json
{
  "patterns": ["/camera"],
  "properties": {
    "context": "default",
    "presentation": "default",
    "view_controller": "native_camera",
    "requires_authentication": true,
    "title": "Take Photo"
  }
}
```

The native app checks for custom properties in its delegate:

```swift
// iOS
func handle(proposal: VisitProposal) -> ProposalResult {
    if let viewController = proposal.properties["view_controller"] as? String {
        switch viewController {
        case "native_camera":
            return .acceptCustom(CameraViewController())
        default:
            return .accept
        }
    }
    return .accept
}
```

```kotlin
// Android
override fun shouldNavigate(proposal: VisitProposal): Boolean {
    val viewController = proposal.properties["view_controller"]
    return when (viewController) {
        "native_camera" -> {
            navigator.route(NativeCameraFragment::class)
            false  // We handled it ourselves
        }
        else -> true  // Let Turbo handle it
    }
}
```

### Pull-to-Refresh Per Path

Enable or disable pull-to-refresh on a per-path basis:

```json
{
  "rules": [
    {
      "patterns": ["/.*"],
      "properties": {
        "pull_to_refresh_enabled": true
      }
    },
    {
      "patterns": ["/new$", "/edit$", "/.*/edit$"],
      "properties": {
        "pull_to_refresh_enabled": false
      }
    },
    {
      "patterns": ["/maps"],
      "properties": {
        "pull_to_refresh_enabled": false
      }
    }
  ]
}
```

Disable pull-to-refresh on:
- Form pages (new, edit) -- accidental pulls would lose form data.
- Map views -- pull gesture conflicts with map panning.
- Pages with their own vertical scroll interactions.

### Serving From Rails

Create a Rails controller that serves the path configuration JSON:

```ruby
# app/controllers/api/v1/turbo/path_configurations_controller.rb
module Api
  module V1
    module Turbo
      class PathConfigurationsController < ApplicationController
        # Skip CSRF and authentication for this endpoint
        skip_before_action :verify_authenticity_token
        skip_before_action :authenticate_user!, if: -> { defined?(authenticate_user!) }

        def show
          render json: path_configuration, content_type: "application/json"
        end

        private

        def path_configuration
          {
            settings: {
              tabs: [
                { title: "Home", path: "/", icon: "house" },
                { title: "Search", path: "/search", icon: "magnifyingglass" },
                { title: "Profile", path: "/profile", icon: "person" }
              ]
            },
            rules: rules
          }
        end

        def rules
          [
            # Default: push web view
            {
              patterns: [".*"],
              properties: {
                context: "default",
                presentation: "default",
                pull_to_refresh_enabled: true
              }
            },
            # Forms open as modals
            {
              patterns: ["/new$", "/edit$"],
              properties: {
                context: "modal",
                presentation: "default",
                pull_to_refresh_enabled: false
              }
            },
            # Native screens
            {
              patterns: ["/camera"],
              properties: {
                context: "default",
                presentation: "default",
                view_controller: "native_camera"
              }
            },
            # Login opens as modal
            {
              patterns: ["/login", "/sign_in"],
              properties: {
                context: "modal",
                presentation: "default",
                pull_to_refresh_enabled: false
              }
            }
          ]
        end
      end
    end
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      namespace :turbo do
        resource :path_configuration, only: :show
      end
    end
  end
end
```

For simpler setups, serve a static JSON file from `public/`:

```
# public/turbo/path_configuration.json
{
  "settings": {},
  "rules": [
    {
      "patterns": [".*"],
      "properties": {
        "context": "default",
        "presentation": "default"
      }
    }
  ]
}
```

### Loading in iOS

Configure path configuration sources in the `TurboNavigator` setup:

```swift
let navigator = TurboNavigator()
navigator.pathConfiguration = PathConfiguration(
    sources: [
        // Local fallback bundled in the app bundle (always available offline)
        .file(Bundle.main.url(forResource: "path-configuration", withExtension: "json")!),
        // Remote source served from Rails (takes priority when available)
        .server(URL(string: "https://app.example.com/api/v1/turbo/path_configuration.json")!)
    ]
)
```

The local file provides routing rules when the app is offline or on first launch before the remote configuration loads. The remote source is fetched asynchronously and merged on top of the local rules when available.

### Loading in Android

```kotlin
// In Application.onCreate() or MainActivity.onCreate()
PathConfiguration.configure(
    assets = listOf("json/path-configuration.json"),  // Local fallback in assets/
    remoteUrls = listOf("https://app.example.com/api/v1/turbo/path_configuration.json")
)
```

Place the local fallback at `app/src/main/assets/json/path-configuration.json`.

### Caching and Updating

Path configuration is cached by the native SDK:

- **iOS**: Cached in `UserDefaults` after first successful remote fetch. Subsequent app launches use the cached version while fetching a fresh copy in the background.
- **Android**: Cached in `SharedPreferences` with the same background refresh strategy.

To force an update, set appropriate cache headers on the Rails response:

```ruby
def show
  # Cache for 5 minutes, then revalidate
  expires_in 5.minutes, public: true
  render json: path_configuration
end
```

For immediate rollout of critical changes, use a short or zero cache duration:

```ruby
def show
  # No caching -- always fetch fresh (use sparingly)
  response.headers["Cache-Control"] = "no-cache, no-store"
  render json: path_configuration
end
```

## Pattern Card

### GOOD: Server-Driven Path Configuration With Local Fallback

```json
{
  "settings": {
    "tabs": [
      { "title": "Home", "path": "/", "icon": "house" },
      { "title": "Profile", "path": "/profile", "icon": "person" }
    ]
  },
  "rules": [
    {
      "patterns": [".*"],
      "properties": {
        "context": "default",
        "presentation": "default",
        "pull_to_refresh_enabled": true
      }
    },
    {
      "patterns": ["/new$", "/edit$"],
      "properties": {
        "context": "modal",
        "presentation": "default",
        "pull_to_refresh_enabled": false
      }
    },
    {
      "patterns": ["/camera"],
      "properties": {
        "view_controller": "native_camera"
      }
    }
  ]
}
```

```swift
// Local fallback + remote source
navigator.pathConfiguration = PathConfiguration(sources: [
    .file(Bundle.main.url(forResource: "path-configuration", withExtension: "json")!),
    .server(URL(string: "https://app.example.com/api/v1/turbo/path_configuration.json")!)
])
```

This approach keeps all routing logic on the server, provides offline fallback, uses regex patterns for flexible matching, and can be updated without an app store release.

### BAD: Hardcoded Native Routing Logic

```swift
// Hardcoded URL-to-presentation mapping in native code
func handle(proposal: VisitProposal) -> ProposalResult {
    let path = proposal.url.path

    // Every new route requires a native app update
    if path.contains("settings") || path.contains("profile/edit") {
        presentModally(proposal)
        return .reject
    }

    if path.contains("camera") {
        return .acceptCustom(CameraViewController())
    }

    if path == "/posts/new" || path == "/comments/new" {
        presentModally(proposal)
        return .reject
    }

    return .accept
}
```

This approach hardcodes every routing decision in native code. Adding a new modal path or changing a presentation style requires a full app store release. There is no offline fallback, no server control, and the logic quickly becomes an unmaintainable chain of string comparisons.
