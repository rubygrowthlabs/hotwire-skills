---
title: "Turbo iOS Setup"
categories:
  - hotwire-native
  - ios
  - turbo
tags:
  - turbo-ios
  - wkwebview
  - turbo-navigator
  - swift
  - xcode
description: >-
  Setting up an iOS app with Hotwire Native (turbo-ios): Xcode project configuration,
  Swift Package Manager dependency, TurboNavigator setup, WKWebView configuration with
  custom user agent, SceneDelegate initialization, and session management for multiple
  navigation stacks.
---

# Turbo iOS Setup

## Table of Contents

- [Overview](#overview)
- [Project Setup](#project-setup)
  - [Swift Package Manager Dependency](#swift-package-manager-dependency)
  - [Minimum Deployment Target](#minimum-deployment-target)
- [Implementation](#implementation)
  - [SceneDelegate Configuration](#scenedelegate-configuration)
  - [TurboNavigator Setup](#turbonavigator-setup)
  - [WKWebView Configuration](#wkwebview-configuration)
  - [Custom User Agent](#custom-user-agent)
  - [Session Management for Multiple Stacks](#session-management-for-multiple-stacks)
  - [Handling Visits and Errors](#handling-visits-and-errors)
- [Pattern Card](#pattern-card)

## Overview

Hotwire Native for iOS (`hotwire-native-ios`) provides a Swift framework that wraps WKWebView with Turbo-aware navigation. Instead of building screens in UIKit or SwiftUI, the native app loads your existing Rails web pages inside a web view and uses Turbo's visit lifecycle to manage navigation transitions natively.

The core component is `TurboNavigator`, which manages a navigation stack of web views. When the user taps a link, Turbo intercepts it, tells the native side to push a new view controller, and the new page loads seamlessly with a native push animation. The result feels like a native app while rendering server-side HTML.

Key concepts:
- **TurboNavigator**: Manages a UINavigationController with web view controllers. Handles push, modal, and replace presentations.
- **Session**: A Turbo session manages a single WKWebView and its visit lifecycle. Each navigation stack gets its own session.
- **PathConfiguration**: A JSON file (served from Rails) that tells the native app how to present each URL pattern.
- **Visit**: A navigation event -- either an "advance" (push) or "replace" (swap current page).

## Project Setup

### Swift Package Manager Dependency

Add the Hotwire Native iOS package to your Xcode project:

1. In Xcode, go to File > Add Package Dependencies.
2. Enter the repository URL: `https://github.com/hotwired/hotwire-native-ios`
3. Select the version rule (e.g., "Up to Next Major Version" from `1.0.0`).
4. Add the `HotwireNative` library to your app target.

Or add it to your `Package.swift` if using a Swift package:

```swift
dependencies: [
    .package(url: "https://github.com/hotwired/hotwire-native-ios", from: "1.0.0")
]
```

### Minimum Deployment Target

Hotwire Native iOS requires iOS 16.0 or later. Set this in your Xcode project's deployment target or in `Package.swift`:

```swift
platforms: [.iOS(.v16)]
```

## Implementation

### SceneDelegate Configuration

The `SceneDelegate` is where the app creates its window and starts the first Turbo visit. Remove the Storyboard reference from `Info.plist` and set up the window programmatically.

```swift
// SceneDelegate.swift
import UIKit
import HotwireNative

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    private lazy var navigator = TurboNavigator()

    func scene(
        _ scene: UIScene,
        willConnectTo session: UISceneSession,
        options connectionOptions: UIScene.ConnectionOptions
    ) {
        guard let windowScene = scene as? UIWindowScene else { return }

        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navigator.rootViewController
        window?.makeKeyAndVisible()

        navigator.route(baseURL.appending(path: "/"))
    }

    private var baseURL: URL {
        // Use localhost for development, production URL for release
        #if DEBUG
        URL(string: "http://localhost:3000")!
        #else
        URL(string: "https://app.example.com")!
        #endif
    }
}
```

### TurboNavigator Setup

`TurboNavigator` is the primary interface for Hotwire Native navigation. It manages a `UINavigationController`, handles path configuration rules, and coordinates Turbo sessions.

```swift
// SceneDelegate.swift (expanded)
import UIKit
import HotwireNative

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    private lazy var navigator: TurboNavigator = {
        let navigator = TurboNavigator()
        navigator.delegate = self
        navigator.pathConfiguration = PathConfiguration(
            sources: [
                // Local fallback bundled with the app
                .file(Bundle.main.url(forResource: "path-configuration", withExtension: "json")!),
                // Remote configuration served from Rails (takes priority)
                .server(baseURL.appending(path: "/api/v1/turbo/path_configuration.json"))
            ]
        )
        return navigator
    }()

    // ... scene setup as above ...
}

extension SceneDelegate: TurboNavigatorDelegate {
    func handle(proposal: VisitProposal) -> ProposalResult {
        // Return .accept to let TurboNavigator handle the visit
        // Return .acceptCustom(controller) to use a custom view controller
        // Return .reject to ignore the visit

        switch proposal.properties["view_controller"] as? String {
        case "native_camera":
            return .acceptCustom(CameraViewController())
        default:
            return .accept
        }
    }

    func visitableDidFailRequest(
        _ visitable: Visitable,
        error: Error,
        retryHandler: RetryHandler
    ) {
        // Handle network errors, show retry UI
        let alert = UIAlertController(
            title: "Connection Error",
            message: "Could not load the page. Please check your connection.",
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "Retry", style: .default) { _ in
            retryHandler.retry()
        })
        alert.addAction(UIAlertAction(title: "Cancel", style: .cancel))
        navigator.rootViewController.present(alert, animated: true)
    }
}
```

### WKWebView Configuration

Customize the WKWebView to set up JavaScript bridge support, cookie sharing, and process pool configuration.

```swift
// WebViewConfiguration.swift
import WebKit
import HotwireNative

extension SceneDelegate {
    func configureWebView() {
        // All Hotwire Native web views share this configuration
        let configuration = WKWebViewConfiguration()

        // Share a single process pool across all web views for cookie sharing
        configuration.processPool = WKProcessPool()

        // Enable inline media playback
        configuration.allowsInlineMediaPlayback = true

        // Configure the web view through Hotwire's configuration point
        Hotwire.config.makeCustomWebView = { config in
            let webView = WKWebView(frame: .zero, configuration: config)
            // Disable link previews (3D Touch / long press)
            webView.allowsLinkPreview = false

            #if DEBUG
            // Enable Web Inspector for debugging in development
            if #available(iOS 16.4, *) {
                webView.isInspectable = true
            }
            #endif

            return webView
        }
    }
}
```

### Custom User Agent

The custom user agent string is critical -- it tells your Rails backend that the request is coming from a Hotwire Native app. Rails uses this to conditionally render native-optimized layouts.

```swift
// In SceneDelegate or AppDelegate, configure before any web view loads
func configureUserAgent() {
    // Hotwire Native appends this to the default WKWebView user agent
    Hotwire.config.applicationUserAgentPrefix = "MyApp/1.0 Turbo Native iOS"
}
```

The resulting user agent string will look like:

```
MyApp/1.0 Turbo Native iOS Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) ...
```

Your Rails backend detects this with the built-in helper:

```ruby
# Returns true when user agent contains "Turbo Native"
turbo_native_app?
```

### Session Management for Multiple Stacks

When using a tab bar, each tab needs its own `TurboNavigator` instance with its own session. This prevents navigation state from leaking between tabs.

```swift
// MainTabBarController.swift
import UIKit
import HotwireNative

class MainTabBarController: UITabBarController {
    private lazy var homeNavigator = makeNavigator()
    private lazy var searchNavigator = makeNavigator()
    private lazy var profileNavigator = makeNavigator()

    override func viewDidLoad() {
        super.viewDidLoad()

        homeNavigator.rootViewController.tabBarItem = UITabBarItem(
            title: "Home",
            image: UIImage(systemName: "house"),
            tag: 0
        )
        searchNavigator.rootViewController.tabBarItem = UITabBarItem(
            title: "Search",
            image: UIImage(systemName: "magnifyingglass"),
            tag: 1
        )
        profileNavigator.rootViewController.tabBarItem = UITabBarItem(
            title: "Profile",
            image: UIImage(systemName: "person"),
            tag: 2
        )

        viewControllers = [
            homeNavigator.rootViewController,
            searchNavigator.rootViewController,
            profileNavigator.rootViewController
        ]

        // Visit the initial URL for each tab
        homeNavigator.route(baseURL.appending(path: "/"))
        searchNavigator.route(baseURL.appending(path: "/search"))
        profileNavigator.route(baseURL.appending(path: "/profile"))
    }

    private func makeNavigator() -> TurboNavigator {
        let navigator = TurboNavigator()
        navigator.delegate = self
        navigator.pathConfiguration = sharedPathConfiguration
        return navigator
    }

    private lazy var sharedPathConfiguration = PathConfiguration(
        sources: [
            .file(Bundle.main.url(forResource: "path-configuration", withExtension: "json")!),
            .server(baseURL.appending(path: "/api/v1/turbo/path_configuration.json"))
        ]
    )

    private var baseURL: URL {
        URL(string: "https://app.example.com")!
    }
}

extension MainTabBarController: TurboNavigatorDelegate {
    func handle(proposal: VisitProposal) -> ProposalResult {
        .accept
    }
}
```

### Handling Visits and Errors

The delegate methods give you fine-grained control over the visit lifecycle:

```swift
extension SceneDelegate: TurboNavigatorDelegate {
    func handle(proposal: VisitProposal) -> ProposalResult {
        // Check path configuration properties to decide how to handle the visit
        let properties = proposal.properties

        // Hand off to a native screen if path config says so
        if let viewController = properties["view_controller"] as? String {
            switch viewController {
            case "native_camera":
                return .acceptCustom(CameraViewController())
            case "native_settings":
                return .acceptCustom(SettingsViewController())
            default:
                break
            }
        }

        // Open external URLs in Safari
        if proposal.url.host != baseURL.host {
            UIApplication.shared.open(proposal.url)
            return .reject
        }

        // Default: let TurboNavigator handle it as a web view
        return .accept
    }

    func sessionDidFinishFormSubmission(_ session: Session) {
        // Called after a form submission completes successfully
        // Good place to provide haptic feedback
        let generator = UINotificationFeedbackGenerator()
        generator.notificationOccurred(.success)
    }
}
```

## Pattern Card

### GOOD: TurboNavigator With Proper Session Config and Path Configuration

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    private lazy var navigator: TurboNavigator = {
        let nav = TurboNavigator()
        nav.delegate = self
        nav.pathConfiguration = PathConfiguration(sources: [
            .file(Bundle.main.url(forResource: "path-configuration", withExtension: "json")!),
            .server(URL(string: "https://app.example.com/api/v1/turbo/path_configuration.json")!)
        ])
        return nav
    }()

    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }
        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navigator.rootViewController
        window?.makeKeyAndVisible()

        Hotwire.config.applicationUserAgentPrefix = "MyApp/1.0 Turbo Native iOS"
        navigator.route(URL(string: "https://app.example.com/")!)
    }
}
```

This approach uses `TurboNavigator` for automatic navigation management, loads path configuration from both a local fallback and a remote server, sets a custom user agent for Rails detection, and delegates native screen decisions to the path configuration JSON.

### BAD: Manual WKWebView Without Turbo Navigation

```swift
class ViewController: UIViewController {
    let webView = WKWebView()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(webView)
        webView.frame = view.bounds

        // No Turbo session -- links open in the same web view with no native transitions
        // No path configuration -- every URL renders identically
        // No custom user agent -- Rails cannot detect native app
        // No error handling -- network failures show a blank white screen
        let request = URLRequest(url: URL(string: "https://app.example.com")!)
        webView.load(request)
    }
}
```

This loses all Hotwire Native benefits: no native navigation transitions, no server-driven routing, no error handling, no session management, and no way for the Rails backend to detect the native app. The user sees a website in a frame rather than a native app experience.
