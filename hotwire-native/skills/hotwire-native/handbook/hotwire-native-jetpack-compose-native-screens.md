# Rendering Native Screens with Jetpack Compose in Hotwire Native
## HotwireFragment, Path Configuration Routing, and MVVM Data Flow

### Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Project Configuration (Compose + Google Maps)](#project-configuration-compose--google-maps)
3. [Fragment Layout (XML)](#fragment-layout-xml)
4. [HotwireFragment with ComposeView (MapFragment)](#hotwire-fragment-with-composeview-mapfragment)
5. [Composable View (MapView)](#composable-view-mapview)
6. [Path Configuration and Fragment Registration](#path-configuration-and-fragment-registration)
7. [Rails JSON Endpoint (shared with iOS)](#rails-json-endpoint-shared-with-ios)
8. [Data Class Model (Hike)](#data-class-model-hike)
9. [ViewModel with mutableStateOf (HikeViewModel)](#viewmodel-with-mutablestateof-hikeviewmodel)
10. [Wiring It All Together](#wiring-it-all-together)

---

## Architecture Overview

Android native screens require six components -- one more than the iOS equivalent (see `handbook/hotwire-native-swiftui-native-screens.md`) because Compose views live inside XML fragment layouts rather than being directly hosted by a controller:

1. **Fragment Layout (XML)** -- Houses the `ComposeView` container alongside an `AppBarLayout` for toolbar/back navigation
2. **HotwireFragment subclass** -- Inflates the layout, bridges into Compose with `setContent {}`, and exposes `navigator.location` for the data URL
3. **@Composable function** -- The declarative UI layer (e.g., `GoogleMap` with markers)
4. **Path Configuration + Fragment Registration** -- A server-driven JSON rule carrying a `uri` property, paired with a `@HotwireDestinationDeepLink` annotation on the fragment
5. **Rails JSON Endpoint** -- Jbuilder template serving structured data (identical to the iOS endpoint)
6. **Model + ViewModel** -- MVVM layer that fetches, parses, and reactively binds data to the composable

For the decision framework on when native screens are worth the added maintenance cost, refer to "When to Go Native vs. Keep Web Views" in the iOS SwiftUI handbook. Those guidelines are platform-agnostic.

### MVVM on Android

Compose follows **MVVM** (Model-View-ViewModel) rather than Rails' MVC:

| MVVM Component | Rails Equivalent | Android Implementation |
|----------------|-----------------|----------------------|
| Model | Rails model | Kotlin `data class` |
| View | Rails view | `@Composable` function |
| ViewModel | Rails controller | `ViewModel` subclass with `mutableStateOf` |

### Application Subclass for Hotwire Configuration

Path configuration loading and fragment destination registration should live in an `Application` subclass rather than an Activity. Android initializes this class once at launch, making it the right place for framework-level setup:

```kotlin
// app/src/main/java/com/example/app/MyApplication.kt
package com.example.app

import android.app.Application
import dev.hotwire.core.config.Hotwire
import dev.hotwire.core.turbo.config.PathConfiguration

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        Hotwire.loadPathConfiguration(
            context = this,
            location = PathConfiguration.Location(
                remoteFileUrl = "$baseURL/configurations/android_v1.json"
            )
        )
    }
}
```

Point `AndroidManifest.xml` at the subclass with `android:name` (the leading dot is required -- it resolves relative to the package):

```xml
<!-- AndroidManifest.xml -->
<application
    android:name=".MyApplication"
    ...>
```

---

## Project Configuration (Compose + Google Maps)

Native Compose screens require upfront Gradle configuration. Google Maps adds further setup for API key management. This section is one-time work -- subsequent native screens skip straight to the Kotlin.

### Compose Configuration

**1. Pin Kotlin 2.0+ and add the Compose Compiler plugin in `libs.versions.toml`:**

```toml
[versions]
kotlin = "2.x.x"

[plugins]
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
```

Tying `version.ref` to `"kotlin"` keeps the Compose Compiler and Kotlin versions in lockstep automatically.

**2. Declare the plugin (without applying) in the project-level `build.gradle.kts`:**

```kotlin
// build.gradle.kts (project-level)
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.compose.compiler) apply false
}
```

**3. Apply it and flip the Compose build feature on in the app-level `build.gradle.kts`:**

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler)
}

android {
    // ...
    kotlinOptions {
        jvmTarget = "17"
    }
    buildFeatures {
        compose = true
    }
}
```

### Google Maps Configuration

Maps Compose requires a Google Maps Platform API key. The recommended approach uses a `secrets.properties` file kept out of version control, read at build time by Google's Secrets Gradle plugin.

**1. Create `secrets.properties`** at the project root (alongside `local.properties`):

```
MAPS_API_KEY=AIzaXXXXXX-XXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Append `secrets.properties` to `.gitignore` immediately.

**2. Apply the Secrets Gradle plugin in `app/build.gradle.kts`:**

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler)
    id("com.google.android.libraries.mapsplatform.secrets-gradle-plugin")
}

secrets {
    propertiesFileName = "secrets.properties"
}
```

**3. Add the plugin's classpath dependency in the project-level `build.gradle.kts`:**

```kotlin
// build.gradle.kts (project-level)
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.compose.compiler) apply false
}

buildscript {
    dependencies {
        classpath(
            "com.google.android.libraries.mapsplatform" +
            ".secrets-gradle-plugin:secrets-gradle-plugin:2.0.1"
        )
    }
}
```

**4. Expose the key to the runtime via `AndroidManifest.xml`:**

```xml
<!-- AndroidManifest.xml -->
<application ...>
    <!-- ... -->
    <meta-data
        android:name="com.google.android.geo.API_KEY"
        android:value="${MAPS_API_KEY}" />
</application>
```

### Complete Dependencies

The full dependency block with Hotwire Native, Compose BOM, Material3, and Maps Compose:

```kotlin
// app/build.gradle.kts
dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.appcompat)
    implementation(libs.material)
    implementation(libs.androidx.activity)
    implementation(libs.androidx.constraintlayout)

    implementation("dev.hotwire:core:1.2.0")
    implementation("dev.hotwire:navigation-fragments:1.2.0")

    implementation(platform("androidx.compose:compose-bom:2025.04.01"))
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("com.google.maps.android:maps-compose:6.1.0")
    debugImplementation("androidx.compose.ui:ui-tooling")

    testImplementation(libs.junit)
    androidTestImplementation(libs.androidx.junit)
    androidTestImplementation(libs.androidx.espresso.core)
}
```

Sync Gradle after all changes.

---

## Fragment Layout (XML)

A `ComposeView` occupies the full screen by default, which means it paints over the toolbar and leaves users with no back button. The fix is wrapping it in a `ConstraintLayout` alongside an `AppBarLayout` that provides the standard Material toolbar:

```xml
<!-- app/src/main/res/layout/fragment_map.xml -->
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <com.google.android.material.appbar.AppBarLayout
        android:id="@+id/app_bar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <com.google.android.material.appbar.MaterialToolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

    </com.google.android.material.appbar.AppBarLayout>

    <androidx.compose.ui.platform.ComposeView
        android:id="@+id/compose_view"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/app_bar" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

Key details:

- `fitsSystemWindows="true"` prevents content from rendering behind the Android status bar.
- The `ComposeView` uses `0dp` height with top/bottom constraints so it fills exactly the space between the toolbar and the bottom edge.
- This layout structure is reusable across all Compose-based native screens -- only the fragment class and composable function change.

---

## HotwireFragment with ComposeView (MapFragment)

`MapFragment` is the glue between Hotwire Native's fragment-based navigation and Jetpack Compose's declarative UI. It inflates the XML layout, locates the `ComposeView`, and hands off rendering to a `@Composable` function via `setContent {}`.

```kotlin
// app/src/main/java/com/example/app/fragments/MapFragment.kt
package com.example.app.fragments

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.ui.platform.ComposeView
import com.example.app.R
import com.example.app.viewmodels.HikeViewModel
import com.google.android.gms.maps.model.CameraPosition
import com.google.android.gms.maps.model.LatLng
import com.google.maps.android.compose.GoogleMap
import com.google.maps.android.compose.MapProperties
import com.google.maps.android.compose.MapType
import com.google.maps.android.compose.Marker
import com.google.maps.android.compose.rememberCameraPositionState
import com.google.maps.android.compose.rememberMarkerState
import dev.hotwire.navigation.destinations.HotwireDestinationDeepLink
import dev.hotwire.navigation.fragments.HotwireFragment

@HotwireDestinationDeepLink(uri = "hotwire://fragment/map")
class MapFragment : HotwireFragment() {
    private lateinit var viewModel: HikeViewModel

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        viewModel = HikeViewModel(url = "${navigator.location}.json")

        val view = inflater.inflate(R.layout.fragment_map, container, false)
        view.findViewById<ComposeView>(R.id.compose_view).apply {
            setContent {
                MapView(viewModel = viewModel)
            }
        }
        return view
    }
}
```

Key details:

- Subclassing `HotwireFragment` provides the `navigator` instance. `navigator.location` holds the current URL (e.g., `/hikes/42/map`) -- append `.json` for the data endpoint.
- `lateinit var` is necessary because the view model depends on `navigator.location`, which isn't available until `onCreateView`.
- `setContent {}` is the single bridge point between Android's view system and Compose -- everything inside the lambda is `@Composable` territory.
- The `@HotwireDestinationDeepLink` annotation declares which `uri` value in path configuration should route to this fragment. The framework handles the matching automatically.

---

## Composable View (MapView)

`MapView` is a standalone `@Composable` function defined outside the `MapFragment` class. It receives the view model, kicks off the data fetch, and renders the map once coordinates arrive:

```kotlin
// Defined at the bottom of MapFragment.kt (outside the class)
@Composable
fun MapView(viewModel: HikeViewModel) {
    LaunchedEffect(viewModel) {
        viewModel.fetchCoordinates()
    }

    val hike = viewModel.hike.value
    if (hike != null) {
        val markerState = rememberMarkerState(position = hike.coordinate)
        val cameraPositionState = rememberCameraPositionState {
            position = CameraPosition.fromLatLngZoom(hike.coordinate, 15f)
        }
        GoogleMap(
            cameraPositionState = cameraPositionState,
            properties = MapProperties(mapType = MapType.HYBRID)
        ) {
            Marker(
                state = markerState,
                title = hike.name
            )
        }
    }
}
```

Key details:

- `LaunchedEffect(viewModel)` fires the network request on first composition -- it serves the same role as SwiftUI's `.task {}` modifier.
- The `remember*` functions (`rememberCameraPositionState`, `rememberMarkerState`) survive recomposition and app backgrounding, preventing the map from snapping back to a default position.
- `MapType.HYBRID` overlays road labels on satellite imagery, matching the iOS `.mapStyle(.hybrid)` appearance.
- The null check on `hike` delays rendering until data arrives -- Compose will recompose automatically when `mutableStateOf` updates.
- `@Preview` is intentionally omitted because Google Maps does not render in Compose previews.

---

## Path Configuration and Fragment Registration

Android requires two things that iOS handles differently: explicit fragment registration at startup, and a `@HotwireDestinationDeepLink` annotation for URI-based routing.

### Register Fragments at Startup

Every fragment that Hotwire Native can navigate to must be registered in the `Application` subclass -- including the default `HotwireWebFragment`:

```kotlin
// app/src/main/java/com/example/app/MyApplication.kt
package com.example.app

import android.app.Application
import com.example.app.fragments.MapFragment
import dev.hotwire.core.config.Hotwire
import dev.hotwire.core.turbo.config.PathConfiguration
import dev.hotwire.navigation.config.registerFragmentDestinations
import dev.hotwire.navigation.fragments.HotwireWebFragment

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        Hotwire.loadPathConfiguration(
            context = this,
            location = PathConfiguration.Location(
                remoteFileUrl = "$baseURL/configurations/android_v1.json"
            )
        )

        Hotwire.registerFragmentDestinations(
            HotwireWebFragment::class,
            MapFragment::class,
        )
    }
}
```

### Server-Side Path Configuration

The `android_v1` path configuration method uses a `uri` property (not `view_controller` as on iOS) to identify native screens:

```ruby
# app/controllers/configurations_controller.rb
class ConfigurationsController < ApplicationController
  def ios_v1
    # ...
  end

  def android_v1
    render json: {
      settings: {},
      rules: [
        {
          patterns: [
            "/new$",
            "/edit$"
          ],
          properties: {
            context: "modal"
          }
        },
        {
          patterns: [
            "/hikes/[0-9]+/map"
          ],
          properties: {
            uri: "hotwire://fragment/map",
            title: "Map"
          }
        }
      ]
    }
  end
end
```

### Platform Routing Comparison

| Aspect | iOS | Android |
|--------|-----|---------|
| Path config property | `view_controller: "map"` | `uri: "hotwire://fragment/map"` |
| Native-side routing | `NavigatorDelegate` switch on `proposal.viewController` | `@HotwireDestinationDeepLink` annotation on fragment |
| Screen registration | Implicit via switch statement | Explicit `Hotwire.registerFragmentDestinations()` call |

iOS routing is imperative -- your `NavigatorDelegate` inspects each proposal and returns the appropriate controller. Android routing is declarative -- the annotation on the fragment class and the `uri` in path configuration are matched by the framework without any manual dispatch code.

For full path configuration details, see `references/path-configuration.md`.

---

## Rails JSON Endpoint (shared with iOS)

Both platforms consume the same Jbuilder template. The view model appends `.json` to the web URL (e.g., `/hikes/1/map.json`), and Rails content negotiation routes to the Jbuilder view:

```ruby
# app/views/maps/show.json.jbuilder
json.extract! @hike, :name, :latitude, :longitude
```

Output:

```json
{
    "name": "Crystal Springs",
    "latitude": 45.479588,
    "longitude": -122.635317
}
```

See the "Rails JSON Endpoint with Jbuilder" section in `handbook/hotwire-native-swiftui-native-screens.md` for additional context on content negotiation and controller setup.

---

## Data Class Model (Hike)

The model mirrors the JSON structure and exposes a computed `coordinate` for the Maps SDK:

```kotlin
// app/src/main/java/com/example/app/models/Hike.kt
package com.example.app.models

import com.google.android.gms.maps.model.LatLng

data class Hike(
    val name: String,
    val latitude: Double,
    val longitude: Double
) {
    val coordinate: LatLng
        get() = LatLng(latitude, longitude)
}
```

Key details:

- `data class` gives you `equals()`, `hashCode()`, `toString()`, and `copy()` for free -- think of it as Ruby's `Struct` with named fields.
- The `coordinate` computed property wraps raw doubles in a `LatLng` (the Android equivalent of `CLLocationCoordinate2D` on iOS).
- Kotlin has no built-in `Decodable` protocol. JSON-to-model mapping happens manually via `JSONObject` in the view model, which is more verbose but avoids adding a serialization library for a single endpoint.

---

## ViewModel with mutableStateOf (HikeViewModel)

The view model owns the data lifecycle: it holds the URL, performs the network fetch on a background dispatcher, parses the JSON response, and publishes the result reactively via `mutableStateOf`:

```kotlin
// app/src/main/java/com/example/app/viewmodels/HikeViewModel.kt
package com.example.app.viewmodels

import androidx.compose.runtime.mutableStateOf
import androidx.lifecycle.ViewModel
import com.example.app.models.Hike
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.json.JSONObject
import java.net.URL

class HikeViewModel(private val url: String) : ViewModel() {
    var hike = mutableStateOf<Hike?>(null)
        private set

    suspend fun fetchCoordinates() {
        try {
            val data = withContext(Dispatchers.IO) {
                URL(url).readText()
            }
            val json = JSONObject(data)
            hike.value = Hike(
                name = json.getString("name"),
                latitude = json.getDouble("latitude"),
                longitude = json.getDouble("longitude")
            )
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
}
```

Key details:

- **`mutableStateOf<Hike?>(null)`** -- Compose's reactive primitive. Any `@Composable` that reads `hike.value` will automatically recompose when the value changes. This fills the same role as `@Observable` on iOS.
- **`private set`** -- Only the view model can write to `hike`; composables treat it as read-only.
- **`suspend fun`** -- Kotlin's equivalent of Swift's `async`. The caller must be in a coroutine scope (provided by `LaunchedEffect` in the composable).
- **`withContext(Dispatchers.IO)`** -- Explicitly moves the network call off the main thread. Unlike Swift where `await` handles thread management, Kotlin requires you to choose the dispatcher.
- **`URL(url).readText()`** -- A one-liner HTTP GET. Sufficient for simple JSON endpoints; production apps may prefer OkHttp or Ktor for retry logic and auth headers.
- **Manual `JSONObject` parsing** -- Each field is extracted by name, unlike iOS where `JSONDecoder` maps keys to `Decodable` properties automatically. For apps with many native endpoints, consider adding Kotlinx Serialization or Moshi.

### Platform Comparison: ViewModel Layer

| Concern | iOS (Swift) | Android (Kotlin) |
|---------|-------------|------------------|
| Reactive binding | `@Observable` macro | `mutableStateOf()` |
| Async marker | `async` | `suspend` |
| HTTP request | `URLSession.shared.data(from:)` | `URL(url).readText()` |
| JSON deserialization | `JSONDecoder` + `Decodable` protocol | Manual `JSONObject` field extraction |
| Threading | Implicit with `await` | Explicit `withContext(Dispatchers.IO)` |
| View-side trigger | `.task {}` SwiftUI modifier | `LaunchedEffect` composable |

---

## Wiring It All Together

End-to-end data flow when a user taps a hike's "Map" link:

```
1. User taps link to /hikes/42/map
          |
2. Hotwire Native evaluates path configuration
   -> pattern "/hikes/[0-9]+/map" matches
   -> uri = "hotwire://fragment/map"
          |
3. Framework resolves @HotwireDestinationDeepLink("hotwire://fragment/map")
   -> MapFragment is instantiated
          |
4. MapFragment.onCreateView() executes
   -> Initializes HikeViewModel with "${navigator.location}.json"
   -> Inflates fragment_map.xml, locates ComposeView
   -> Calls setContent { MapView(viewModel) }
          |
5. MapView composes for the first time
   -> LaunchedEffect fires viewModel.fetchCoordinates()
          |
6. HikeViewModel fetches /hikes/42/map.json on Dispatchers.IO
   -> Rails responds with Jbuilder JSON: { name, latitude, longitude }
          |
7. JSONObject parses response, creates Hike data class
   -> hike.value is updated via mutableStateOf
          |
8. Compose detects state change and recomposes MapView
   -> GoogleMap renders with Marker at the hike's coordinates
```

### File Structure

```
android/app/src/main/
├── java/com/example/app/
|   ├── MyApplication.kt               # Hotwire config + fragment registration
|   ├── activities/
|   |   └── MainActivity.kt            # Main activity
|   ├── fragments/
|   |   └── MapFragment.kt             # HotwireFragment + @Composable MapView
|   ├── models/
|   |   └── Hike.kt                    # Data class model
|   └── viewmodels/
|       └── HikeViewModel.kt           # ViewModel with mutableStateOf
└── res/layout/
    └── fragment_map.xml                # ComposeView + AppBarLayout

rails/app/
├── controllers/
|   └── configurations_controller.rb   # Path config with uri rule (android_v1)
└── views/maps/
    └── show.json.jbuilder              # JSON endpoint (shared with iOS)
```

### Adding More Native Screens

Each additional native screen follows the same recipe:

1. Create a layout XML pairing `AppBarLayout` with `ComposeView`
2. Subclass `HotwireFragment` and annotate with `@HotwireDestinationDeepLink`
3. Define a `@Composable` function for the screen's UI
4. Register the new fragment in `Hotwire.registerFragmentDestinations()` within the `Application` subclass
5. Add a path configuration rule whose `uri` matches the annotation
6. Add a Jbuilder endpoint if the screen consumes server data
7. Create a `data class` and `ViewModel` with `mutableStateOf` for data fetching
