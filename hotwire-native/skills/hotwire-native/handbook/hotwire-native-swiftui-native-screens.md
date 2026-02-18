# Rendering Native Screens with SwiftUI in Hotwire Native
## UIHostingController, Path Configuration Routing, and MVVM Data Flow

### Table of Contents
1. [When to Go Native vs. Keep Web Views](#when-to-go-native-vs-keep-web-views)
2. [Architecture Overview](#architecture-overview)
3. [SwiftUI View (MapView)](#swiftui-view-mapview)
4. [UIHostingController Bridge (MapController)](#uihostingcontroller-bridge-mapcontroller)
5. [Path Configuration Routing](#path-configuration-routing)
6. [NavigatorDelegate and ProposalResult](#navigatordelegate-and-proposalresult)
7. [Rails JSON Endpoint with Jbuilder](#rails-json-endpoint-with-jbuilder)
8. [Model with Decodable (Hike)](#model-with-decodable-hike)
9. [ViewModel with @Observable (HikeViewModel)](#viewmodel-with-observable-hikeviewmodel)
10. [Wiring It All Together](#wiring-it-all-together)

---

## When to Go Native vs. Keep Web Views

A major benefit of Hotwire Native is being able to drop down into native Swift when needed. This unlocks access to the latest iOS APIs and provides a way to render fully native screens. However, each native screen adds complexity and maintenance, so reserve them for the highlights of your app.

### Go Native When

| Scenario | Why Native Wins |
|----------|----------------|
| Home screen / launch screen | Launch quickly with highest fidelity; cache data for offline access |
| Maps with gestures | Swiping, pinching, and panning work as expected with native MapKit |
| Platform API integration | HealthKit, ARKit, CoreML data flows directly from API to Swift without HTML round trips |
| Tabbed content screens | 37signals uses native screens for each tab in Basecamp |

### Keep Using Web Views When

| Scenario | Why Web Wins |
|----------|-------------|
| Settings and preferences | Change frequently; web updates are cheap vs. native + API changes |
| Checkout flows | Adding coupon codes or new fields requires no app store release |
| CRUD operations | Not unique to your product experience; time is better spent on critical native workflows |
| Dynamic heterogeneous content | News feeds with mixed item types require each type as its own native view; new types need app store releases |

**Rule of thumb:** A SwiftUI or Jetpack Compose update often requires updates to both the view and the API on the server. Each API change needs backward compatibility with all previous app versions. Web changes avoid this entirely.

---

## Architecture Overview

Adding a SwiftUI native screen to a Hotwire Native app requires five components:

1. **SwiftUI View** -- Renders the native UI (e.g., a `Map` with markers)
2. **UIHostingController** -- Bridges SwiftUI into UIKit so Hotwire Native can route to it
3. **Path Configuration** -- Server-driven JSON rule that tells the client when to display the native screen
4. **Rails JSON Endpoint** -- Serves structured data for the native view via Jbuilder
5. **Model + ViewModel** -- Fetches, parses, and binds data to the view using MVVM

Unlike MVC in Rails, SwiftUI uses the **MVVM** (Model-View-ViewModel) pattern:

| MVVM Component | Rails Equivalent | Responsibility |
|----------------|-----------------|----------------|
| Model | Rails model | Data objects (`Decodable` structs) |
| View | Rails view | Renders content to the screen (SwiftUI) |
| ViewModel | Rails controller | Coordinates data flow between model and view |

---

## SwiftUI View (MapView)

Create a SwiftUI view under `App/Views/`. Import MapKit and render a `Map` with markers driven by the view model.

```swift
// App/Views/MapView.swift
import MapKit
import SwiftUI

struct MapView: View {
    var viewModel: HikeViewModel

    var body: some View {
        Map {
            if let hike = viewModel.hike {
                Marker(hike.name, coordinate: hike.coordinate)
            }
        }
        .mapStyle(.hybrid(elevation: .realistic))
        .navigationTitle("Map")
        .clipped()
        .task {
            await viewModel.fetchCoordinates()
        }
    }
}

#Preview {
    let url = URL(string: "https://example.com")!
    let viewModel = HikeViewModel(url: url)
    viewModel.hike = Hike(
        name: "Mount Tabor",
        latitude: 45.511881,
        longitude: -122.595706
    )
    return MapView(viewModel: viewModel)
}
```

Key details:

- `.task {}` is a SwiftUI modifier that runs async code when the view loads -- this triggers the network request.
- `.mapStyle(.hybrid(elevation: .realistic))` renders satellite imagery with realistic terrain.
- `.clipped()` prevents the map from bleeding into the navigation or tab bar.
- The `if let` safely unwraps the optional `hike` so the marker only renders after data arrives.

---

## UIHostingController Bridge (MapController)

SwiftUI views cannot be used directly with Hotwire Native's UIKit-based navigation. `UIHostingController` bridges the gap between the two frameworks.

```swift
// App/Controllers/MapController.swift
import SwiftUI
import UIKit

class MapController: UIHostingController<MapView> {
    convenience init(url: URL) {
        let viewModel = HikeViewModel(url: url)
        let view = MapView(viewModel: viewModel)
        self.init(rootView: view)
    }
}
```

Key details:

- `UIHostingController<MapView>` is typed to render a specific SwiftUI view.
- The `convenience init(url:)` accepts the URL from the Hotwire Native proposal, creates the view model and view, then delegates to the designated initializer.
- **Designated vs. convenience initializers in Swift:** Designated initializers are the primary entry points (like `initialize` in Ruby). Convenience initializers are secondary helpers (like a `self.create_from_url(url)` class method) that call a designated initializer within the same class.

---

## Path Configuration Routing

Path configuration is a server-served JSON file that tells the native client how to present each URL. Add a rule that matches the hike map path and assigns the `view_controller` property.

For full path configuration details, see `references/path-configuration.md`.

```ruby
# app/controllers/configurations_controller.rb
class ConfigurationsController < ApplicationController
  def ios_v1
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
            view_controller: "map"
          }
        }
      ]
    }
  end
end
```

The `view_controller` property is recognized by Hotwire Native iOS. The framework exposes it through the `NavigatorDelegate` when the user taps a link matching that pattern.

---

## NavigatorDelegate and ProposalResult

The `NavigatorDelegate` protocol's `handle(proposal:from:)` method is called every time the user taps a link. It returns a `ProposalResult` that determines what screen to render.

### ProposalResult Cases

| Case | Behavior |
|------|----------|
| `.accept` | Route a standard `VisitableViewController` (web view) |
| `.acceptCustom(UIViewController)` | Route a custom view controller (native screen) |
| `.reject` | Cancel the navigation; no screen change |

### Implementation

Wire up the `NavigatorDelegate` on your `SceneDelegate` and switch on the `viewController` property from path configuration:

```swift
// App/Delegates/SceneDelegate.swift
import HotwireNative
import UIKit

let baseURL = URL(string: "http://localhost:3000")!

class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    private lazy var tabBarController = HotwireTabBarController(
        navigatorDelegate: self
    )

    // ...
}

extension SceneDelegate: NavigatorDelegate {
    func handle(
        proposal: VisitProposal,
        from navigator: Navigator
    ) -> ProposalResult {
        switch proposal.viewController {
        case "map": .acceptCustom(MapController(url: proposal.url))
        default: .accept
        }
    }
}
```

Key details:

- The `tabBarController` must use `lazy var` (not `let`) because it references `self` as the delegate.
- `proposal.viewController` reads the `view_controller` property from path configuration.
- `proposal.url` is the URL the user tapped, passed to `MapController` for data fetching.
- Add additional `case` entries as you build more native screens.

For native navigation patterns including tabs, modals, and deep links, see `references/native-navigation.md`.

---

## Rails JSON Endpoint with Jbuilder

The native screen needs structured data from the server. Create a Jbuilder view that renders when the URL is requested with a `.json` extension.

```ruby
# app/views/maps/show.json.jbuilder
json.extract! @hike, :name, :latitude, :longitude
```

This produces:

```json
{
    "name": "Crystal Springs",
    "latitude": 45.479588,
    "longitude": -122.635317
}
```

The native view model appends `.json` to the original web URL (e.g., `/hikes/1/map.json`), so the same controller action serves both the HTML web view and the JSON native endpoint via Rails content negotiation.

---

## Model with Decodable (Hike)

Create a Swift struct conforming to `Decodable` that mirrors the JSON response. Add a computed property to convert latitude/longitude into a `CLLocationCoordinate2D` for MapKit.

```swift
// App/Models/Hike.swift
import MapKit

struct Hike: Decodable {
    let name: String
    let latitude: Double
    let longitude: Double

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(
            latitude: latitude,
            longitude: longitude
        )
    }
}
```

`Decodable` allows `JSONDecoder` to parse the JSON response directly into a `Hike` instance with a single line of code -- no manual key mapping needed when property names match the JSON keys.

---

## ViewModel with @Observable (HikeViewModel)

The view model coordinates the data flow: it builds the URL, fetches JSON from the network, and parses it into a model. The `@Observable` macro automatically triggers SwiftUI view redraws when properties change.

```swift
// App/ViewModels/HikeViewModel.swift
import Foundation

@Observable class HikeViewModel {
    var hike: Hike?

    private let url: URL

    init(url: URL) {
        self.url = url.appendingPathExtension("json")
    }

    func fetchCoordinates() async {
        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            hike = try JSONDecoder().decode(Hike.self, from: data)
        } catch {
            print("Failed to fetch hike: \(error.localizedDescription)")
        }
    }
}
```

Key details:

- **URL construction:** `.appendingPathExtension("json")` converts `/hikes/1/map` to `/hikes/1/map.json`, triggering the Jbuilder view on the server.
- **`@Observable` macro:** Binds public properties to the SwiftUI view. When `hike` is set after the network response, SwiftUI automatically redraws the `MapView`. Requires iOS 17+. For older versions, use `ObservableObject` protocol with `@Published` property wrapper.
- **`async`/`await`:** Swift's concurrency model. Unlike Ruby where HTTP requests block, Swift requires explicit `await` to yield the thread during network calls.
- **Error handling:** `do`/`try`/`catch` maps to Ruby's `begin`/`rescue`. Functions that can raise errors must be called with `try`.
- **Ignored tuple element:** `data(from:)` returns `(Data, URLResponse)`. The underscore ignores the response, like `_` in Ruby destructuring.

---

## Wiring It All Together

The complete data flow when a user taps a hike's "Map" link:

```
1. User taps link to /hikes/42/map
          ↓
2. Hotwire Native fetches path configuration
   → matches "/hikes/[0-9]+/map"
   → sets view_controller = "map"
          ↓
3. NavigatorDelegate.handle(proposal:) is called
   → proposal.viewController == "map"
   → returns .acceptCustom(MapController(url: proposal.url))
          ↓
4. MapController creates HikeViewModel with the URL
   → HikeViewModel appends .json extension
          ↓
5. MapView renders and .task {} fires
   → viewModel.fetchCoordinates() called
          ↓
6. HikeViewModel fetches /hikes/42/map.json
   → Rails serves Jbuilder JSON: { name, latitude, longitude }
          ↓
7. JSONDecoder parses response into Hike model
   → viewModel.hike is set
          ↓
8. @Observable triggers SwiftUI redraw
   → Map renders with Marker at hike's coordinates
```

### File Structure

```
ios/App/
├── Controllers/
│   └── MapController.swift          # UIHostingController bridge
├── Delegates/
│   └── SceneDelegate.swift          # NavigatorDelegate routing
├── Models/
│   └── Hike.swift                   # Decodable data model
├── ViewModels/
│   └── HikeViewModel.swift          # @Observable data coordinator
└── Views/
    └── MapView.swift                # SwiftUI map view

rails/app/
├── controllers/
│   └── configurations_controller.rb # Path configuration with view_controller rule
└── views/maps/
    └── show.json.jbuilder           # JSON endpoint for native screens
```

### Adding More Native Screens

To add another native screen, repeat the pattern:

1. Create a SwiftUI view for the UI
2. Wrap it in a `UIHostingController` subclass with a `convenience init(url:)`
3. Add a path configuration rule with a `view_controller` property
4. Add a `case` to the `NavigatorDelegate` switch statement
5. Create a Jbuilder endpoint if the screen needs server data
6. Add a `Decodable` model and `@Observable` view model if fetching data