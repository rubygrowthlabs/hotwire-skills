---
title: "Turbo Android Setup"
categories:
  - hotwire-native
  - android
  - turbo
tags:
  - turbo-android
  - webview
  - turbo-activity
  - kotlin
  - gradle
description: >-
  Setting up an Android app with Hotwire Native (turbo-android): Gradle dependency,
  TurboActivity and TurboSessionNavHostFragment configuration, WebView setup with
  custom user agent, navigation graph for fragment-based navigation, back navigation,
  and deep link handling.
---

# Turbo Android Setup

## Table of Contents

- [Overview](#overview)
- [Project Setup](#project-setup)
  - [Gradle Dependency](#gradle-dependency)
  - [Minimum SDK Version](#minimum-sdk-version)
  - [Internet Permission](#internet-permission)
- [Implementation](#implementation)
  - [MainActivity Configuration](#mainactivity-configuration)
  - [TurboSessionNavHostFragment Setup](#turbosessionnavhostfragment-setup)
  - [Navigation Graph](#navigation-graph)
  - [WebView Configuration](#webview-configuration)
  - [Custom User Agent](#custom-user-agent)
  - [Handling Back Navigation](#handling-back-navigation)
  - [Deep Link Handling](#deep-link-handling)
- [Pattern Card](#pattern-card)

## Overview

Hotwire Native for Android (`hotwire-native-android`) provides a Kotlin framework that wraps Android's WebView with Turbo-aware navigation using the Jetpack Navigation component. Instead of building screens with Jetpack Compose or XML layouts, the native app loads your Rails web pages inside WebView fragments and uses Turbo's visit lifecycle to drive fragment-based navigation transitions.

The architecture follows Android conventions:
- **TurboActivity**: A single Activity that hosts a navigation graph of fragments.
- **TurboSessionNavHostFragment**: A NavHostFragment that manages the Turbo session and WebView.
- **TurboWebFragment**: A fragment that displays a web page via Turbo. One per screen in the navigation stack.
- **PathConfiguration**: The same JSON routing rules used on iOS, controlling how each URL is presented (push, modal, replace, native).

Key differences from iOS:
- Android uses a single Activity with fragments, rather than multiple UIViewControllers.
- Navigation is managed by Jetpack Navigation component with a nav graph XML.
- Back navigation uses the Android system back gesture/button and `OnBackPressedDispatcher`.

## Project Setup

### Gradle Dependency

Add the Hotwire Native Android dependency to your app-level `build.gradle.kts`:

```kotlin
// app/build.gradle.kts
dependencies {
    implementation("dev.hotwire:hotwire-native-android:1.0.0")
}
```

Also ensure you have the Jetpack Navigation dependency:

```kotlin
dependencies {
    implementation("dev.hotwire:hotwire-native-android:1.0.0")
    implementation("androidx.navigation:navigation-fragment-ktx:2.7.7")
    implementation("androidx.navigation:navigation-ui-ktx:2.7.7")
}
```

### Minimum SDK Version

Hotwire Native Android requires API level 28 (Android 9.0) or higher:

```kotlin
// app/build.gradle.kts
android {
    defaultConfig {
        minSdk = 28
    }
}
```

### Internet Permission

Add the internet permission to `AndroidManifest.xml`:

```xml
<!-- AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET" />
    <!-- ... -->
</manifest>
```

## Implementation

### MainActivity Configuration

The main activity hosts the navigation graph and serves as the entry point. It extends `TurboActivity` which handles session lifecycle and navigation events.

```kotlin
// MainActivity.kt
package com.example.myapp

import android.os.Bundle
import dev.hotwire.core.turbo.activities.TurboActivity
import dev.hotwire.core.turbo.config.PathConfiguration
import dev.hotwire.core.turbo.session.TurboSessionNavHostFragment

class MainActivity : TurboActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Configure path configuration sources
        PathConfiguration.configure(
            assets = listOf("json/path-configuration.json"),  // Local fallback
            remoteUrls = listOf("$baseUrl/api/v1/turbo/path_configuration.json")  // Remote
        )
    }

    override fun onProvideNavHostFragment(): TurboSessionNavHostFragment {
        return supportFragmentManager
            .findFragmentById(R.id.main_nav_host) as TurboSessionNavHostFragment
    }

    companion object {
        const val baseUrl = "https://app.example.com"
    }
}
```

### TurboSessionNavHostFragment Setup

Create a custom `TurboSessionNavHostFragment` that configures the Turbo session and navigation behavior:

```kotlin
// MainSessionNavHostFragment.kt
package com.example.myapp

import dev.hotwire.core.turbo.config.TurboPathConfiguration
import dev.hotwire.core.turbo.session.TurboSessionNavHostFragment

class MainSessionNavHostFragment : TurboSessionNavHostFragment() {

    override val sessionName: String
        get() = "main"

    override val startLocation: String
        get() = MainActivity.baseUrl

    override val registeredFragments: List<KClass<out Fragment>>
        get() = listOf(
            WebFragment::class,
            WebModalFragment::class,
            NativeCameraFragment::class
        )

    override val registeredActivities: List<KClass<out AppCompatActivity>>
        get() = emptyList()

    override fun onSessionCreated() {
        super.onSessionCreated()

        // Configure custom user agent
        session.webView.settings.apply {
            userAgentString = "$userAgentString MyApp/1.0 Turbo Native Android"
            javaScriptEnabled = true
            domStorageEnabled = true
        }
    }
}
```

The activity layout hosts the nav host fragment:

```xml
<!-- res/layout/activity_main.xml -->
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/main_nav_host"
        android:name="com.example.myapp.MainSessionNavHostFragment"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:defaultNavHost="true" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

### Navigation Graph

Define the navigation graph with fragment destinations. Hotwire Native uses this to map URL patterns to fragments:

```xml
<!-- res/navigation/main_nav_graph.xml -->
<?xml version="1.0" encoding="utf-8"?>
<navigation
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/main_nav_graph"
    app:startDestination="@id/webFragment">

    <!-- Default web view fragment for all URLs -->
    <fragment
        android:id="@+id/webFragment"
        android:name="com.example.myapp.WebFragment"
        android:label="Web">
    </fragment>

    <!-- Modal web view fragment -->
    <fragment
        android:id="@+id/webModalFragment"
        android:name="com.example.myapp.WebModalFragment"
        android:label="Modal">
    </fragment>

    <!-- Native camera fragment -->
    <fragment
        android:id="@+id/nativeCameraFragment"
        android:name="com.example.myapp.NativeCameraFragment"
        android:label="Camera">
    </fragment>

</navigation>
```

Create the web fragment that displays Turbo web pages:

```kotlin
// WebFragment.kt
package com.example.myapp

import dev.hotwire.core.turbo.fragments.TurboWebFragment
import dev.hotwire.core.turbo.nav.TurboNavGraphDestination

@TurboNavGraphDestination(uri = "turbo://fragment/web")
class WebFragment : TurboWebFragment() {

    override fun onVisitCompleted(location: String, completedOffline: Boolean) {
        super.onVisitCompleted(location, completedOffline)
        // Page loaded successfully -- update toolbar title if needed
        toolbarForNavigation()?.title = title()
    }

    override fun onVisitErrorReceived(location: String, errorCode: Int) {
        super.onVisitErrorReceived(location, errorCode)
        when (errorCode) {
            401 -> navigator.route("${MainActivity.baseUrl}/login")
            else -> showErrorView(errorCode)
        }
    }
}
```

### WebView Configuration

Configure the WebView for optimal Hotwire Native behavior:

```kotlin
// In MainSessionNavHostFragment.onSessionCreated()
override fun onSessionCreated() {
    super.onSessionCreated()

    session.webView.settings.apply {
        // Required for Hotwire Native
        javaScriptEnabled = true
        domStorageEnabled = true

        // Enable caching for offline support
        cacheMode = WebSettings.LOAD_DEFAULT
        databaseEnabled = true

        // Allow mixed content if needed during development
        if (BuildConfig.DEBUG) {
            mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
        }

        // Custom user agent for Rails detection
        userAgentString = "$userAgentString MyApp/1.0 Turbo Native Android"
    }

    // Enable WebView debugging in development
    if (BuildConfig.DEBUG) {
        WebView.setWebContentsDebuggingEnabled(true)
    }
}
```

### Custom User Agent

The user agent string must include "Turbo Native" for Rails backend detection. Append to the existing user agent rather than replacing it:

```kotlin
// The user agent is appended, not replaced
session.webView.settings.userAgentString =
    "${session.webView.settings.userAgentString} MyApp/1.0 Turbo Native Android"
```

The resulting user agent looks like:

```
Mozilla/5.0 (Linux; Android 14; Pixel 8) AppleWebKit/537.36 ... MyApp/1.0 Turbo Native Android
```

Your Rails backend detects this with:

```ruby
# Returns true when user agent contains "Turbo Native"
turbo_native_app?
```

### Handling Back Navigation

Android back navigation integrates with the Jetpack Navigation component. Hotwire Native handles most cases automatically, but you can customize:

```kotlin
// WebFragment.kt
@TurboNavGraphDestination(uri = "turbo://fragment/web")
class WebFragment : TurboWebFragment() {

    override fun onBackPressed() {
        // Check if the web view has its own back history
        if (session.webView.canGoBack()) {
            session.webView.goBack()
        } else {
            // Pop the fragment from the navigation stack
            navigator.pop()
        }
    }
}
```

For modal fragments, dismiss instead of popping:

```kotlin
// WebModalFragment.kt
@TurboNavGraphDestination(uri = "turbo://fragment/web/modal")
class WebModalFragment : TurboWebBottomSheetDialogFragment() {

    override fun onBackPressed() {
        dismiss()
    }
}
```

### Deep Link Handling

Configure deep links in `AndroidManifest.xml` to open specific URLs directly in the app:

```xml
<!-- AndroidManifest.xml -->
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />

        <data
            android:scheme="https"
            android:host="app.example.com"
            android:pathPrefix="/" />
    </intent-filter>
</activity>
```

Handle the deep link in the Activity:

```kotlin
// MainActivity.kt
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    // Handle deep link intent
    intent?.data?.let { uri ->
        val location = uri.toString()
        navigator.route(location)
    }
}

override fun onNewIntent(intent: Intent?) {
    super.onNewIntent(intent)
    // Handle deep link when app is already running
    intent?.data?.let { uri ->
        navigator.route(uri.toString())
    }
}
```

## Pattern Card

### GOOD: TurboActivity With Proper Navigation and Session Config

```kotlin
class MainActivity : TurboActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        PathConfiguration.configure(
            assets = listOf("json/path-configuration.json"),
            remoteUrls = listOf("$baseUrl/api/v1/turbo/path_configuration.json")
        )
    }

    override fun onProvideNavHostFragment(): TurboSessionNavHostFragment {
        return supportFragmentManager
            .findFragmentById(R.id.main_nav_host) as MainSessionNavHostFragment
    }
}

class MainSessionNavHostFragment : TurboSessionNavHostFragment() {
    override val sessionName = "main"
    override val startLocation = "https://app.example.com"

    override val registeredFragments = listOf(
        WebFragment::class,
        WebModalFragment::class
    )

    override fun onSessionCreated() {
        super.onSessionCreated()
        session.webView.settings.userAgentString =
            "${session.webView.settings.userAgentString} MyApp/1.0 Turbo Native Android"
    }
}
```

This approach uses `TurboActivity` with a navigation graph for native transitions, configures path configuration from both local and remote sources, registers typed fragments for web and modal presentations, and sets a custom user agent for Rails detection.

### BAD: Plain WebView Activity Without Turbo

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val webView = WebView(this)
        setContentView(webView)

        webView.settings.javaScriptEnabled = true

        // No Turbo session -- all navigation happens inside a single WebView
        // No path configuration -- cannot present modals or native screens
        // No custom user agent -- Rails cannot detect native app
        // No fragment navigation -- no native back stack
        // No error handling -- network errors show Android's default error page
        webView.loadUrl("https://app.example.com")
    }

    // Back button just calls webView.goBack() with no stack management
    override fun onBackPressed() {
        val webView = findViewById<WebView>(android.R.id.content)
        if (webView.canGoBack()) webView.goBack() else super.onBackPressed()
    }
}
```

This loses all Hotwire Native benefits: no native navigation transitions between pages, no server-driven routing, no modal presentations, no typed fragment registration, and no way for the Rails backend to detect the native app. Every page renders identically in a single WebView with browser-like forward/back behavior instead of native stack navigation.
