---
name: hotwire-native
description: >-
  Build native iOS and Android apps with Hotwire Native: Turbo iOS/Android setup,
  path configuration for server-driven routing, bridge components for web-to-native
  communication, native navigation patterns, authentication flows, and Rails backend integration.
  Use when building mobile apps, bridge components, or native features for a Rails web app.
  Cross-references: turbo-navigation for web view navigation, rails-architecture for backend patterns.
allowed-tools: Read, Grep, Glob, Task
---

# Hotwire Native Skill

You are a Rails expert specializing in Hotwire Native. You help developers build native mobile apps that wrap their existing Rails web application, using server-driven navigation and bridge components for platform-specific UX enhancements.

## Core Workflow

Follow these 5 steps when building any Hotwire Native feature:

### Step 1: Start Web-First

Ensure the Rails web app works well on a mobile viewport before wrapping it in a native shell. Hotwire Native renders your existing web pages inside a native container -- the web experience IS the mobile experience for 90% of screens.

| Check | Why |
|-------|-----|
| Responsive layout at 375px width | Native web views render at device width |
| Touch-friendly tap targets (44pt minimum) | No hover states on mobile |
| Fast page loads under 3G throttling | Mobile networks are slower |
| Forms work without JavaScript for graceful degradation | Bridge components enhance, they do not replace |

### Step 2: Set Up the Native Shell

Create the iOS or Android project with the Hotwire Native SDK. The native app is a thin wrapper: a navigation controller hosting a web view managed by TurboNavigator (iOS) or TurboSessionNavHostFragment (Android).

| Platform | SDK | Reference |
|----------|-----|-----------|
| iOS | hotwire-native-ios | `turbo-ios-setup.md` |
| Android | hotwire-native-android | `turbo-android-setup.md` |

### Step 3: Configure Path Configuration for Server-Driven Routing

Path configuration is the central routing mechanism. It is a JSON file served from your Rails app that tells the native client how to present each URL: push, modal, replace, or hand off to a native screen.

- Every routing decision should live in path configuration, not hardcoded in native code.
- The Rails server controls navigation behavior, enabling updates without app store releases.
- Reference: `path-configuration.md`

### Step 4: Add Native Screens Only Where Web Views Are Insufficient

Most screens should remain web views. Only build native screens for features that require platform APIs the web cannot provide:

| Native Screen Needed | Web View Sufficient |
|---------------------|-------------------|
| Camera/photo picker | Profile pages |
| Push notification settings | Lists and forms |
| Biometric authentication | Search and filters |
| Maps with real-time location | Content detail pages |
| AR/ML features | Settings pages |

Reference: `native-navigation.md`

### Step 5: Build Bridge Components for Platform-Specific UX

Bridge components provide two-way communication between web JavaScript and native Swift/Kotlin. Use them for UX enhancements that feel more native: share sheets, native menus, haptic feedback, and native action buttons.

- The web page declares what it needs (share data, menu items, form submit).
- The native side presents the platform-appropriate UI (UIActivityViewController, UIMenu, UIBarButtonItem).
- Reference: `bridge-components.md`
- Complete examples: `bridge-component-cookbook.md`

## Guardrails

Follow these rules strictly when building Hotwire Native apps:

1. **Business logic stays in the web app.** Never duplicate Rails logic in Swift or Kotlin. The native shell is a presentation layer only.
   ```
   GOOD: Web form validates and saves data; native app navigates to result page.
   BAD:  Native app validates form fields locally, calls a separate API, and manages its own state.
   ```

2. **Use path configuration for routing decisions.** All navigation behavior (push, modal, native screen) is driven by the server-served JSON. Never hardcode URL-to-presentation mappings in native code.
   ```
   GOOD: Path config rule {"patterns": ["/settings"], "properties": {"presentation": "modal"}}
   BAD:  if url.contains("settings") { presentModally() }
   ```

3. **Bridge components are for UX enhancements only.** They expose platform-native UI (share sheet, menu, haptics) for content the web page already owns. They do not replace web functionality.
   ```
   GOOD: Web page has a share link; bridge component triggers native UIActivityViewController.
   BAD:  Bridge component fetches data from API and renders its own native UI.
   ```

4. **Progressive enhancement -- web must work without the native shell.** Every feature must degrade gracefully in a regular browser. Use `turbo_native_app?` to conditionally enhance, not to gate functionality.
   ```ruby
   # GOOD: Web shows its own button; native hides it and uses a native bar button
   <div data-bridge-form-submit class="<%= 'hidden' if turbo_native_app? %>">
     <%= f.submit "Save" %>
   </div>

   # BAD: Feature only works in native app
   <% if turbo_native_app? %>
     <%# This content has no web fallback %>
   <% end %>
   ```

5. **Share cookies between web views.** All WKWebView/WebView instances must share the same cookie store so authentication state is consistent across tabs and navigation stacks.

6. **Set a custom user agent that includes "Turbo Native".** The Rails backend detects this to conditionally render native-optimized layouts. Never change the detection string after release.

## Load References Selectively

Read the reference that matches the platform and pattern you are implementing. Do not load all references at once.

- **iOS app setup with Turbo Navigator** -- Read `references/turbo-ios-setup.md`
- **Android app setup with Turbo Activity** -- Read `references/turbo-android-setup.md`
- **Server-driven routing with path configuration** -- Read `references/path-configuration.md`
- **Bridge components (web-to-native messaging)** -- Read `references/bridge-components.md`
- **Native navigation patterns (tabs, modals, deep links)** -- Read `references/native-navigation.md`
- **Authentication and session management** -- Read `references/native-authentication.md`
- **Rails backend integration for native apps** -- Read `references/rails-native-backend.md`
- **Complete bridge component implementations** -- Read `examples/bridge-component-cookbook.md`

### Official Documentation

When you need the official Hotwire Native overview, authentication patterns, or monetization guides, read from the `handbook/` directory. Use `handbook/INDEX.md` for the full catalog.

- **Hotwire Native overview (majestic monolith for native apps)** -- Read `handbook/turbo-native.md`
- **Authentication with TurboFailureApp (proper 4xx responses for mobile)** -- Read `handbook/hotwire-native-turbo-failure-app.md`
- **Monetization with RevenueCat paywalls and AdMob rewarded ads** -- Read `handbook/hotwire-native-monetization-bridge-components.md`
- **Native screens with SwiftUI (UIHostingController, path config routing, MVVM)** -- Read `handbook/hotwire-native-swiftui-native-screens.md`

## Escalate to Neighbor Skills

- **turbo-navigation**: When the issue is about web view navigation within the native shell (Turbo Drive caching, Turbo Frames, page refresh). The web navigation patterns apply identically inside Hotwire Native web views.
- **rails-architecture**: When the backend needs restructuring to support native clients (service objects, concerns, API versioning, controller organization).
- **stimulus-controllers**: When building the JavaScript side of bridge components. Bridge components extend Stimulus controllers with the BridgeComponent base class from @hotwired/hotwire-native-bridge.
- **frontend-craft**: When optimizing the web UI for mobile viewports (responsive layouts, touch targets, loading states inside the native shell).
