---
title: "Native Navigation Patterns"
categories:
  - hotwire-native
  - navigation
  - ui
tags:
  - tab-bar
  - modal
  - deep-links
  - native-screens
  - pull-to-refresh
  - navigation-bar
description: >-
  Native navigation patterns for Hotwire Native apps: tab bar navigation with multiple
  TurboNavigator instances, native screens for platform APIs, bottom sheets and modal
  presentations, pull-to-refresh configuration, external URL handling, deep links, and
  navigation bar customization.
---

# Native Navigation Patterns

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Tab Bar Navigation](#tab-bar-navigation)
  - [Native Screens for Platform APIs](#native-screens-for-platform-apis)
  - [Bottom Sheets and Modal Presentation](#bottom-sheets-and-modal-presentation)
  - [Pull-to-Refresh Configuration](#pull-to-refresh-configuration)
  - [Handling External URLs](#handling-external-urls)
  - [Deep Link Handling](#deep-link-handling)
  - [Navigation Bar Customization](#navigation-bar-customization)
- [Pattern Card](#pattern-card)

## Overview

Hotwire Native apps combine web view navigation with native UI chrome. The majority of screens are web views managed by Turbo, but the native shell provides tab bars, navigation bars, modals, and native screens for platform-specific features.

The key principle is that navigation decisions are server-driven through path configuration. The native app reads the path configuration JSON to determine how to present each URL: push onto the navigation stack, present as a modal, hand off to a native screen, or open externally.

Navigation patterns fall into three categories:
1. **Web navigation**: Standard Turbo page visits rendered in web views with native push/pop transitions.
2. **Hybrid navigation**: Web content presented with native chrome (tab bar, navigation bar buttons, pull-to-refresh).
3. **Native navigation**: Fully native screens for features that require platform APIs (camera, maps, biometrics).

## Implementation

### Tab Bar Navigation

A tab bar gives the app a familiar native feel. Each tab has its own `TurboNavigator` (iOS) or `TurboSessionNavHostFragment` (Android) with an independent navigation stack and web view session.

**iOS Implementation:**

```swift
// MainTabBarController.swift
import UIKit
import HotwireNative

class MainTabBarController: UITabBarController {
    private let baseURL = URL(string: "https://app.example.com")!

    private lazy var homeNavigator = makeNavigator()
    private lazy var discoverNavigator = makeNavigator()
    private lazy var notificationsNavigator = makeNavigator()
    private lazy var profileNavigator = makeNavigator()

    override func viewDidLoad() {
        super.viewDidLoad()
        configureTabs()
        visitInitialPages()
    }

    private func configureTabs() {
        homeNavigator.rootViewController.tabBarItem = UITabBarItem(
            title: "Home",
            image: UIImage(systemName: "house"),
            selectedImage: UIImage(systemName: "house.fill")
        )
        discoverNavigator.rootViewController.tabBarItem = UITabBarItem(
            title: "Discover",
            image: UIImage(systemName: "magnifyingglass"),
            selectedImage: UIImage(systemName: "magnifyingglass")
        )
        notificationsNavigator.rootViewController.tabBarItem = UITabBarItem(
            title: "Notifications",
            image: UIImage(systemName: "bell"),
            selectedImage: UIImage(systemName: "bell.fill")
        )
        profileNavigator.rootViewController.tabBarItem = UITabBarItem(
            title: "Profile",
            image: UIImage(systemName: "person"),
            selectedImage: UIImage(systemName: "person.fill")
        )

        viewControllers = [
            homeNavigator.rootViewController,
            discoverNavigator.rootViewController,
            notificationsNavigator.rootViewController,
            profileNavigator.rootViewController
        ]
    }

    private func visitInitialPages() {
        homeNavigator.route(baseURL.appending(path: "/"))
        discoverNavigator.route(baseURL.appending(path: "/discover"))
        notificationsNavigator.route(baseURL.appending(path: "/notifications"))
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
}

extension MainTabBarController: TurboNavigatorDelegate {
    func handle(proposal: VisitProposal) -> ProposalResult {
        // Handle native screen routing
        if let viewController = proposal.properties["view_controller"] as? String {
            switch viewController {
            case "native_camera":
                return .acceptCustom(CameraViewController())
            default:
                break
            }
        }
        return .accept
    }
}
```

**Updating badge counts from the server:**

```swift
// Use a bridge component or Turbo Stream to update badge counts
func updateNotificationBadge(count: Int) {
    notificationsNavigator.rootViewController.tabBarItem.badgeValue =
        count > 0 ? "\(count)" : nil
}
```

**Android Implementation:**

```kotlin
// MainActivity.kt with bottom navigation
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val navHostFragment = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as TurboSessionNavHostFragment

        // Connect bottom navigation to Jetpack Navigation
        val bottomNav = binding.bottomNavigation
        NavigationUI.setupWithNavController(bottomNav, navHostFragment.navController)
    }
}
```

```xml
<!-- res/layout/activity_main.xml -->
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/nav_host_fragment"
        android:name="com.example.myapp.MainSessionNavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        app:defaultNavHost="true" />

    <com.google.android.material.bottomnavigation.BottomNavigationView
        android:id="@+id/bottom_navigation"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:menu="@menu/bottom_navigation" />

</LinearLayout>
```

```xml
<!-- res/menu/bottom_navigation.xml -->
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/homeFragment"
        android:icon="@drawable/ic_home"
        android:title="Home" />
    <item
        android:id="@+id/discoverFragment"
        android:icon="@drawable/ic_search"
        android:title="Discover" />
    <item
        android:id="@+id/notificationsFragment"
        android:icon="@drawable/ic_notifications"
        android:title="Notifications" />
    <item
        android:id="@+id/profileFragment"
        android:icon="@drawable/ic_person"
        android:title="Profile" />
</menu>
```

### Native Screens for Platform APIs

Some features require native screens because web views cannot access platform APIs. Use path configuration to route specific URLs to native view controllers or fragments.

**Path configuration:**

```json
{
  "rules": [
    {
      "patterns": ["/camera"],
      "properties": {
        "view_controller": "native_camera",
        "presentation": "modal"
      }
    },
    {
      "patterns": ["/maps/.*"],
      "properties": {
        "view_controller": "native_map"
      }
    },
    {
      "patterns": ["/settings/notifications"],
      "properties": {
        "view_controller": "native_notification_settings"
      }
    }
  ]
}
```

**iOS native screen:**

```swift
// CameraViewController.swift
import UIKit
import AVFoundation

class CameraViewController: UIViewController {
    private let captureSession = AVCaptureSession()

    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Take Photo"
        navigationItem.leftBarButtonItem = UIBarButtonItem(
            barButtonSystemItem: .cancel,
            target: self,
            action: #selector(cancelTapped)
        )
        setupCamera()
    }

    @objc private func cancelTapped() {
        dismiss(animated: true)
    }

    private func setupCamera() {
        guard let camera = AVCaptureDevice.default(for: .video),
              let input = try? AVCaptureDeviceInput(device: camera) else {
            return
        }
        captureSession.addInput(input)
        // Set up preview layer and capture output...
    }

    func photoWasCaptured(imageURL: URL) {
        // Navigate back to the web view with the photo URL
        // The web view will handle uploading via Active Storage
        dismiss(animated: true) {
            // Post notification or use delegate to pass data back
            NotificationCenter.default.post(
                name: .photoCaptured,
                object: nil,
                userInfo: ["url": imageURL]
            )
        }
    }
}
```

**Android native screen:**

```kotlin
// NativeCameraFragment.kt
@TurboNavGraphDestination(uri = "turbo://fragment/native/camera")
class NativeCameraFragment : Fragment() {

    private val cameraLauncher = registerForActivityResult(
        ActivityResultContracts.TakePicture()
    ) { success ->
        if (success) {
            // Photo captured, navigate back with result
            findNavController().previousBackStackEntry
                ?.savedStateHandle
                ?.set("photo_uri", photoUri.toString())
            findNavController().popBackStack()
        }
    }

    private lateinit var photoUri: Uri

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        launchCamera()
    }

    private fun launchCamera() {
        photoUri = createTempPhotoUri()
        cameraLauncher.launch(photoUri)
    }
}
```

### Bottom Sheets and Modal Presentation

Use path configuration to present specific URLs as modals or bottom sheets:

**Path configuration for modals:**

```json
{
  "rules": [
    {
      "patterns": ["/new$", "/edit$"],
      "properties": {
        "context": "modal",
        "presentation": "default"
      }
    },
    {
      "patterns": ["/share$", "/actions$"],
      "properties": {
        "context": "modal",
        "presentation": "default"
      }
    }
  ]
}
```

**iOS modal presentation styles:**

```swift
// Customize modal presentation in the TurboNavigator delegate
func handle(proposal: VisitProposal) -> ProposalResult {
    // TurboNavigator automatically reads context from path config
    // Modals are presented as page sheets by default on iOS 15+
    // Customize if needed:
    return .accept
}
```

**Android bottom sheet:**

```kotlin
// WebModalFragment.kt
@TurboNavGraphDestination(uri = "turbo://fragment/web/modal")
class WebModalFragment : TurboWebBottomSheetDialogFragment() {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        // Configure bottom sheet behavior
        val bottomSheet = dialog as? BottomSheetDialog
        bottomSheet?.behavior?.apply {
            state = BottomSheetBehavior.STATE_EXPANDED
            isDraggable = true
            skipCollapsed = true
        }
    }
}
```

### Pull-to-Refresh Configuration

Pull-to-refresh is configured per-path in the path configuration JSON:

```json
{
  "rules": [
    {
      "patterns": [".*"],
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

When pull-to-refresh is triggered, Hotwire Native performs a Turbo visit to reload the current page. The web view re-renders with fresh server content.

### Handling External URLs

When a link points to a URL outside your app's domain, open it in the system browser:

**iOS:**

```swift
func handle(proposal: VisitProposal) -> ProposalResult {
    let baseHost = URL(string: "https://app.example.com")!.host

    // Open external URLs in Safari
    if proposal.url.host != baseHost {
        UIApplication.shared.open(proposal.url)
        return .reject
    }

    // Handle mailto:, tel:, etc.
    if proposal.url.scheme != "http" && proposal.url.scheme != "https" {
        UIApplication.shared.open(proposal.url)
        return .reject
    }

    return .accept
}
```

**Android:**

```kotlin
override fun shouldNavigate(proposal: VisitProposal): Boolean {
    val baseHost = "app.example.com"

    if (proposal.url.host != baseHost) {
        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(proposal.url.toString()))
        startActivity(intent)
        return false
    }

    return true
}
```

### Deep Link Handling

Deep links allow external sources (push notifications, emails, other apps) to open specific screens in your Hotwire Native app.

**iOS (Universal Links):**

```swift
// SceneDelegate.swift
func scene(
    _ scene: UIScene,
    continue userActivity: NSUserActivity
) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else { return }

    // Route the deep link URL through TurboNavigator
    navigator.route(url)
}
```

Configure in `Associated Domains` entitlement:
```
applinks:app.example.com
```

Serve the Apple App Site Association file from your Rails app:

```ruby
# config/routes.rb
get "/.well-known/apple-app-site-association", to: "well_known#apple_app_site_association"
```

```ruby
# app/controllers/well_known_controller.rb
class WellKnownController < ApplicationController
  def apple_app_site_association
    render json: {
      applinks: {
        apps: [],
        details: [
          {
            appID: "TEAM_ID.com.example.myapp",
            paths: ["*"]
          }
        ]
      }
    }
  end
end
```

**Android (App Links):**

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="https"
            android:host="app.example.com" />
    </intent-filter>
</activity>
```

Serve the Digital Asset Links file:

```ruby
# config/routes.rb
get "/.well-known/assetlinks.json", to: "well_known#asset_links"
```

```ruby
# app/controllers/well_known_controller.rb
def asset_links
  render json: [
    {
      relation: ["delegate_permission/common.handle_all_urls"],
      target: {
        namespace: "android_app",
        package_name: "com.example.myapp",
        sha256_cert_fingerprints: [ENV["ANDROID_SHA256_FINGERPRINT"]]
      }
    }
  ]
end
```

### Navigation Bar Customization

Customize the native navigation bar (title, buttons, appearance) based on the web page content.

**iOS:**

```swift
// In TurboNavigatorDelegate or a custom Visitable
func visitableDidRender(visitable: Visitable) {
    // Title is automatically set from the <title> tag
    // Customize further:
    visitable.visitableViewController.navigationItem.largeTitleDisplayMode = .never

    // Add right bar button from bridge component (see strada-bridge-components.md)
}
```

**Setting title from the web page:**

Hotwire Native automatically reads the `<title>` tag and sets the navigation bar title. Override with a `data-turbo-title` attribute on the `<body>`:

```erb
<%# app/views/posts/show.html.erb %>
<% content_for :title, @post.title %>
```

```erb
<%# app/views/layouts/application.html.erb %>
<title><%= content_for(:title) || "My App" %></title>
```

## Pattern Card

### GOOD: Tab-Based Navigation With Path Config and Native Screens

```swift
class MainTabBarController: UITabBarController, TurboNavigatorDelegate {
    private lazy var homeNavigator = makeNavigator()
    private lazy var profileNavigator = makeNavigator()

    override func viewDidLoad() {
        super.viewDidLoad()
        viewControllers = [
            homeNavigator.rootViewController,
            profileNavigator.rootViewController
        ]
        homeNavigator.route(baseURL.appending(path: "/"))
        profileNavigator.route(baseURL.appending(path: "/profile"))
    }

    func handle(proposal: VisitProposal) -> ProposalResult {
        // Server-driven: path config decides presentation
        // Native screens only for platform APIs
        if proposal.properties["view_controller"] as? String == "native_camera" {
            return .acceptCustom(CameraViewController())
        }
        return .accept
    }
}
```

Each tab has its own session and navigation stack. Path configuration drives all routing decisions. Native screens are reserved for platform APIs that web views cannot access. Tab bar provides familiar native navigation chrome around web content.

### BAD: Single WebView Without Native Navigation

```swift
class ViewController: UIViewController {
    let webView = WKWebView()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.addSubview(webView)
        webView.frame = view.bounds

        // Single web view, no tabs, no native navigation
        // User gets browser-like experience with no native feel
        // No back stack management -- back gesture goes to previous web page, not previous screen
        // No modals -- everything renders in the same web view
        // No path configuration -- every URL renders identically
        webView.load(URLRequest(url: URL(string: "https://app.example.com")!))
    }
}
```

This approach renders the entire app in a single web view with no native chrome. There is no tab bar, no native back stack, no modal presentations, and no way to route to native screens. The user experience is indistinguishable from a bookmark on the home screen.
