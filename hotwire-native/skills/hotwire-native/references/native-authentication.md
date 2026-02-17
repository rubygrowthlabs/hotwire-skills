---
title: "Native Authentication"
categories:
  - hotwire-native
  - authentication
  - security
tags:
  - authentication
  - cookies
  - keychain
  - session
  - biometric
  - wkwebview
description: >-
  Authentication in Hotwire Native apps: cookie-based session management in WKWebView,
  sharing cookies between web view and native HTTP client, Keychain persistence for
  auth tokens, login flow with 401 detection, session expiry and re-authentication
  handling, and biometric authentication integration.
---

# Native Authentication

## Table of Contents

- [Overview](#overview)
- [Authentication Architecture](#authentication-architecture)
- [Implementation](#implementation)
  - [Cookie-Based Session Management](#cookie-based-session-management)
  - [Sharing Cookies Between Web View and Native HTTP Client](#sharing-cookies-between-web-view-and-native-http-client)
  - [Keychain Storage for Persistent Auth](#keychain-storage-for-persistent-auth)
  - [Login Flow: Detecting 401 and Presenting Login](#login-flow-detecting-401-and-presenting-login)
  - [Handling Session Expiry and Re-Authentication](#handling-session-expiry-and-re-authentication)
  - [Biometric Authentication Integration](#biometric-authentication-integration)
- [Pattern Card](#pattern-card)

## Overview

Authentication in Hotwire Native apps follows a simple principle: the web app manages authentication, and the native shell stays in sync. Since Hotwire Native renders your Rails web pages in a web view, the same cookie-based session that works in a browser works identically in the native app.

The native shell's responsibilities are:
1. Detect when the session has expired (401 response or redirect to login).
2. Present the login screen (web view or native) so the user can re-authenticate.
3. Persist auth tokens in the Keychain for session restoration across app launches.
4. Optionally integrate biometric authentication (Face ID, Touch ID, fingerprint) for quick unlock.

The native shell does NOT manage its own auth state, call a separate login API, or maintain JWT tokens independently of the web session. The web view's cookies are the single source of truth.

## Authentication Architecture

```
App Launch
    │
    ├── Check Keychain for stored session cookie
    │   │
    │   ├── Found → Set cookie in WKWebView cookie store → Navigate to home
    │   │                                                      │
    │   │                                           ┌──────────┴──────────┐
    │   │                                           │ Cookie valid?       │
    │   │                                           ├── Yes → App ready   │
    │   │                                           └── No (401) → Login  │
    │   │
    │   └── Not found → Navigate to login page
    │
    └── Login page (web view)
        │
        ├── User submits credentials (standard Rails form)
        │
        ├── Rails sets session cookie
        │
        ├── WKWebView cookie store receives cookie
        │
        ├── Native app reads cookie and stores in Keychain
        │
        └── Navigate to home page
```

## Implementation

### Cookie-Based Session Management

WKWebView manages cookies through `WKHTTPCookieStore`. Cookies set by the Rails server during login are automatically available for all subsequent web view requests.

```swift
// CookieManager.swift
import WebKit

class CookieManager {
    static let shared = CookieManager()

    private let cookieStore: WKHTTPCookieStore

    init() {
        // Use the default data store's cookie store
        // This is shared across all WKWebViews using the default configuration
        self.cookieStore = WKWebsiteDataStore.default().httpCookieStore
    }

    /// Get all cookies for the app's domain
    func getSessionCookies() async -> [HTTPCookie] {
        let cookies = await cookieStore.allCookies()
        return cookies.filter { $0.domain.contains("app.example.com") }
    }

    /// Set a cookie in the web view's cookie store
    func setCookie(_ cookie: HTTPCookie) async {
        await cookieStore.setCookie(cookie)
    }

    /// Clear all cookies (logout)
    func clearAllCookies() async {
        let cookies = await cookieStore.allCookies()
        for cookie in cookies {
            await cookieStore.deleteCookie(cookie)
        }
    }

    /// Observe cookie changes
    func observeCookieChanges(handler: @escaping () -> Void) {
        // WKHTTPCookieStore does not provide a change observer,
        // so check after each page load instead
    }
}
```

Ensure all WKWebView instances share the same data store:

```swift
// In your WebView configuration
func configureWebView() {
    Hotwire.config.makeCustomWebView = { config in
        // Use the default data store (shared cookie jar)
        // Do NOT create a new WKWebsiteDataStore.nonPersistent()
        // as that creates an isolated cookie jar
        let webView = WKWebView(frame: .zero, configuration: config)
        return webView
    }
}
```

### Sharing Cookies Between Web View and Native HTTP Client

If your native app needs to make HTTP requests outside the web view (e.g., uploading a photo captured natively), share the session cookie:

```swift
// NativeHTTPClient.swift
import Foundation

class NativeHTTPClient {
    static let shared = NativeHTTPClient()

    private let session: URLSession

    init() {
        let config = URLSessionConfiguration.default
        // Share cookies with WKWebView by using the same cookie storage
        config.httpCookieStorage = HTTPCookieStorage.shared
        self.session = URLSession(configuration: config)
    }

    /// Sync web view cookies to URLSession's cookie storage
    func syncCookiesFromWebView() async {
        let webViewCookies = await CookieManager.shared.getSessionCookies()
        for cookie in webViewCookies {
            HTTPCookieStorage.shared.setCookie(cookie)
        }
    }

    /// Make an authenticated request using the web view's session
    func upload(imageData: Data, to url: URL) async throws -> Data {
        // Sync cookies before making the request
        await syncCookiesFromWebView()

        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/octet-stream", forHTTPHeaderField: "Content-Type")

        // Include the same user agent so Rails recognizes this as a Turbo Native request
        request.setValue(
            "MyApp/1.0 Turbo Native iOS",
            forHTTPHeaderField: "User-Agent"
        )

        let (data, response) = try await session.upload(for: request, from: imageData)

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw AuthError.requestFailed
        }

        return data
    }
}

enum AuthError: Error {
    case requestFailed
    case sessionExpired
    case biometricFailed
}
```

**Android cookie sharing:**

```kotlin
// CookieSync.kt
import android.webkit.CookieManager

object CookieSyncHelper {
    private val cookieManager = CookieManager.getInstance()

    /**
     * Get the session cookie from the WebView's cookie store
     * for use in native HTTP requests (e.g., OkHttp).
     */
    fun getSessionCookie(baseUrl: String): String? {
        return cookieManager.getCookie(baseUrl)
    }

    /**
     * Add cookies to an OkHttp request by reading from WebView's store.
     */
    fun addCookiesToRequest(
        requestBuilder: okhttp3.Request.Builder,
        baseUrl: String
    ): okhttp3.Request.Builder {
        val cookies = getSessionCookie(baseUrl)
        if (cookies != null) {
            requestBuilder.addHeader("Cookie", cookies)
        }
        return requestBuilder
    }

    /**
     * Clear all cookies (logout).
     */
    fun clearCookies() {
        cookieManager.removeAllCookies(null)
        cookieManager.flush()
    }
}
```

### Keychain Storage for Persistent Auth

Store the session cookie in the iOS Keychain so the session persists across app launches. WKWebView cookies can be cleared by the system, so Keychain is the reliable persistence layer.

```swift
// KeychainSessionStore.swift
import Security
import Foundation

class KeychainSessionStore {
    static let shared = KeychainSessionStore()

    private let serviceName = "com.example.myapp.session"
    private let accountName = "session_cookie"

    /// Save the session cookie data to Keychain
    func saveSession(cookies: [HTTPCookie]) {
        guard let data = try? NSKeyedArchiver.archivedData(
            withRootObject: cookies.map { $0.properties },
            requiringSecureCoding: false
        ) else { return }

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: serviceName,
            kSecAttrAccount as String: accountName,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
        ]

        // Delete existing item first
        SecItemDelete(query as CFDictionary)

        // Add new item
        SecItemAdd(query as CFDictionary, nil)
    }

    /// Restore session cookies from Keychain
    func restoreSession() -> [HTTPCookie]? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: serviceName,
            kSecAttrAccount as String: accountName,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess,
              let data = result as? Data,
              let propertiesArray = try? NSKeyedUnarchiver.unarchivedObject(
                  ofClasses: [NSArray.self, NSDictionary.self, NSString.self, NSNumber.self, NSDate.self],
                  from: data
              ) as? [[HTTPCookiePropertyKey: Any]] else {
            return nil
        }

        return propertiesArray.compactMap { HTTPCookie(properties: $0) }
    }

    /// Clear session from Keychain (logout)
    func clearSession() {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: serviceName,
            kSecAttrAccount as String: accountName
        ]
        SecItemDelete(query as CFDictionary)
    }
}
```

**Restoring session on app launch:**

```swift
// SceneDelegate.swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
    guard let windowScene = scene as? UIWindowScene else { return }

    window = UIWindow(windowScene: windowScene)
    window?.rootViewController = navigator.rootViewController
    window?.makeKeyAndVisible()

    // Restore session cookies from Keychain
    Task {
        if let cookies = KeychainSessionStore.shared.restoreSession() {
            for cookie in cookies {
                await CookieManager.shared.setCookie(cookie)
            }
            // Navigate to home -- cookies will be sent with the request
            navigator.route(baseURL.appending(path: "/"))
        } else {
            // No stored session, go to login
            navigator.route(baseURL.appending(path: "/login"))
        }
    }
}
```

**Save session after successful login:**

```swift
// In TurboNavigatorDelegate
func visitableDidRender(visitable: Visitable) {
    // After any successful page load, save cookies to Keychain
    Task {
        let cookies = await CookieManager.shared.getSessionCookies()
        if !cookies.isEmpty {
            KeychainSessionStore.shared.saveSession(cookies: cookies)
        }
    }
}
```

### Login Flow: Detecting 401 and Presenting Login

When the server returns a 401 (Unauthorized) or redirects to the login page, the native app should present the login screen as a modal:

**Path configuration for login:**

```json
{
  "rules": [
    {
      "patterns": ["/login", "/sign_in", "/users/sign_in"],
      "properties": {
        "context": "modal",
        "presentation": "replace_root",
        "pull_to_refresh_enabled": false
      }
    }
  ]
}
```

**iOS 401 handling:**

```swift
extension SceneDelegate: TurboNavigatorDelegate {
    func visitableDidFailRequest(
        _ visitable: Visitable,
        error: Error,
        retryHandler: RetryHandler
    ) {
        if let turboError = error as? TurboError,
           case .httpFailure(let statusCode) = turboError,
           statusCode == 401 {
            // Session expired -- present login
            presentLogin(retryHandler: retryHandler)
            return
        }

        // Handle other errors...
    }

    private func presentLogin(retryHandler: RetryHandler?) {
        // Clear stale session
        KeychainSessionStore.shared.clearSession()

        // Navigate to login -- path config will present as modal
        navigator.route(baseURL.appending(path: "/login"))

        // Store retry handler to retry the failed request after login
        self.pendingRetryHandler = retryHandler
    }

    // Called after login form submission succeeds
    func sessionDidFinishFormSubmission(_ session: Session) {
        // Save new session cookies
        Task {
            let cookies = await CookieManager.shared.getSessionCookies()
            KeychainSessionStore.shared.saveSession(cookies: cookies)
        }

        // Retry the request that triggered the 401
        pendingRetryHandler?.retry()
        pendingRetryHandler = nil
    }
}
```

**Android 401 handling:**

```kotlin
// WebFragment.kt
override fun onVisitErrorReceived(location: String, errorCode: Int) {
    when (errorCode) {
        401 -> {
            // Navigate to login screen
            navigator.route("${MainActivity.baseUrl}/login")
        }
        else -> super.onVisitErrorReceived(location, errorCode)
    }
}
```

### Handling Session Expiry and Re-Authentication

Rails may redirect expired sessions to the login page rather than returning 401. Detect this by checking the URL after a visit completes:

```swift
func visitableDidRender(visitable: Visitable) {
    guard let currentURL = visitable.visitableURL else { return }

    // If we landed on the login page unexpectedly, the session expired
    let loginPaths = ["/login", "/sign_in", "/users/sign_in"]
    if loginPaths.contains(where: { currentURL.path.hasSuffix($0) }) {
        // Session expired -- clear Keychain and let the user log in
        KeychainSessionStore.shared.clearSession()
    }
}
```

For background session refresh (keeping the session alive while the app is backgrounded):

```swift
// AppDelegate.swift
func applicationDidBecomeActive(_ application: UIApplication) {
    // Ping the server to check if the session is still valid
    Task {
        await NativeHTTPClient.shared.syncCookiesFromWebView()
        let url = URL(string: "https://app.example.com/api/v1/session/check")!
        var request = URLRequest(url: url)
        request.httpMethod = "GET"

        if let (_, response) = try? await URLSession.shared.data(for: request),
           let httpResponse = response as? HTTPURLResponse,
           httpResponse.statusCode == 401 {
            // Session expired while app was backgrounded
            NotificationCenter.default.post(name: .sessionExpired, object: nil)
        }
    }
}
```

### Biometric Authentication Integration

Add Face ID or Touch ID as a quick-unlock mechanism. The biometric check protects the Keychain-stored session, not the web login.

```swift
// BiometricAuthManager.swift
import LocalAuthentication

class BiometricAuthManager {
    static let shared = BiometricAuthManager()

    var isBiometricAvailable: Bool {
        let context = LAContext()
        var error: NSError?
        return context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error)
    }

    func authenticate(reason: String) async -> Bool {
        let context = LAContext()
        context.localizedFallbackTitle = "Use Password"

        do {
            return try await context.evaluatePolicy(
                .deviceOwnerAuthenticationWithBiometrics,
                localizedReason: reason
            )
        } catch {
            return false
        }
    }
}
```

**Integrating biometric unlock into the app launch flow:**

```swift
// SceneDelegate.swift
func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
    guard let windowScene = scene as? UIWindowScene else { return }

    window = UIWindow(windowScene: windowScene)
    window?.rootViewController = navigator.rootViewController
    window?.makeKeyAndVisible()

    Task {
        // Check for stored session
        guard let cookies = KeychainSessionStore.shared.restoreSession(),
              !cookies.isEmpty else {
            navigator.route(baseURL.appending(path: "/login"))
            return
        }

        // If biometric is available and enabled by user, require it
        let biometricEnabled = UserDefaults.standard.bool(forKey: "biometric_unlock_enabled")
        if biometricEnabled && BiometricAuthManager.shared.isBiometricAvailable {
            let authenticated = await BiometricAuthManager.shared.authenticate(
                reason: "Unlock MyApp"
            )
            guard authenticated else {
                // Biometric failed -- show login instead
                navigator.route(baseURL.appending(path: "/login"))
                return
            }
        }

        // Restore cookies and navigate
        for cookie in cookies {
            await CookieManager.shared.setCookie(cookie)
        }
        navigator.route(baseURL.appending(path: "/"))
    }
}
```

**Android biometric authentication:**

```kotlin
// BiometricHelper.kt
import androidx.biometric.BiometricPrompt
import androidx.core.content.ContextCompat
import androidx.fragment.app.FragmentActivity

class BiometricHelper(private val activity: FragmentActivity) {

    fun authenticate(onSuccess: () -> Unit, onFailure: () -> Unit) {
        val executor = ContextCompat.getMainExecutor(activity)
        val callback = object : BiometricPrompt.AuthenticationCallback() {
            override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                onSuccess()
            }
            override fun onAuthenticationFailed() {
                onFailure()
            }
            override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                onFailure()
            }
        }

        val biometricPrompt = BiometricPrompt(activity, executor, callback)
        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Unlock MyApp")
            .setSubtitle("Authenticate to continue")
            .setNegativeButtonText("Use password")
            .build()

        biometricPrompt.authenticate(promptInfo)
    }
}
```

## Pattern Card

### GOOD: Cookie Sync With Keychain Persistence and 401 Detection

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate, TurboNavigatorDelegate {
    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options: UIScene.ConnectionOptions) {
        // Restore session from Keychain on launch
        Task {
            if let cookies = KeychainSessionStore.shared.restoreSession() {
                for cookie in cookies {
                    await CookieManager.shared.setCookie(cookie)
                }
                navigator.route(baseURL.appending(path: "/"))
            } else {
                navigator.route(baseURL.appending(path: "/login"))
            }
        }
    }

    func visitableDidFailRequest(_ visitable: Visitable, error: Error, retryHandler: RetryHandler) {
        // Detect 401 and present login
        if case .httpFailure(401) = error as? TurboError {
            KeychainSessionStore.shared.clearSession()
            navigator.route(baseURL.appending(path: "/login"))
        }
    }

    func sessionDidFinishFormSubmission(_ session: Session) {
        // Save new cookies after login
        Task {
            let cookies = await CookieManager.shared.getSessionCookies()
            KeychainSessionStore.shared.saveSession(cookies: cookies)
        }
    }
}
```

This approach uses the web app's cookie-based session as the single source of truth. Keychain provides persistence across app launches. The 401 detection automatically redirects to login. After successful login, cookies are saved for next launch. The web app's authentication logic is never duplicated in native code.

### BAD: Separate Native Auth System Disconnected From Web Session

```swift
class AuthManager {
    // BAD: Managing auth tokens separately from the web view
    var jwtToken: String?

    func login(email: String, password: String) async throws {
        // BAD: Calling a separate API endpoint instead of using the web login form
        let url = URL(string: "https://app.example.com/api/auth/login")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.httpBody = try JSONEncoder().encode(["email": email, "password": password])

        let (data, _) = try await URLSession.shared.data(for: request)
        let response = try JSONDecoder().decode(AuthResponse.self, from: data)
        jwtToken = response.token

        // BAD: JWT token is completely separate from web view cookies
        // The web view has no session -- user sees login page inside the app
    }

    func makeRequest(to url: URL) async throws -> Data {
        var request = URLRequest(url: url)
        // BAD: Adding JWT header that the web view knows nothing about
        request.setValue("Bearer \(jwtToken!)", forHTTPHeaderField: "Authorization")
        let (data, _) = try await URLSession.shared.data(for: request)
        return data
    }
}
```

This approach creates a parallel authentication system disconnected from the web view. The native app manages JWT tokens while the web view uses cookies -- they are never in sync. The user may be authenticated in native HTTP requests but see a login page in the web view. This defeats the entire purpose of Hotwire Native, which is to render the existing web app inside a native shell.
