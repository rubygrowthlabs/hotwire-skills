# Monetizing Hotwire Native Apps with Bridge Components
## RevenueCat Paywalls and AdMob Rewarded Ads

### Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Web Side -- Rails + Stimulus](#2-web-side----rails--stimulus)
3. [Native Side -- iOS Swift](#3-native-side----ios-swift)
4. [Ad-Free Timer and Subscription State](#4-ad-free-timer-and-subscription-state)
5. [Configuration (xcconfig to Runtime)](#5-configuration-xcconfig-to-runtime)
6. [Integration Checklist](#6-integration-checklist)

---

## 1. Architecture Overview

Two bridge components handle monetization in a Hotwire Native app: `purchase` (RevenueCat subscriptions) and `rewarded-ad` (AdMob rewarded video). Rails renders all UI; native handles all SDK calls. No JavaScript API requests. No token passing.

### How It Works

1. Rails renders a promo card partial with `display: none`
2. The `rewarded-ad` bridge controller sends a `connect` event to native
3. Native checks RevenueCat `customerInfo` for active entitlements plus a temporary ad-free timer
4. Non-subscribers see the card revealed with two CTAs: **Subscribe** and **Watch Ad**
5. Subscribe triggers RevenueCat's paywall via the `purchase` bridge component
6. Watch Ad presents an AdMob rewarded video via the `rewarded-ad` bridge component
7. Successful ad completion grants a 2-hour ad-free period; banners disappear immediately

### Flow Diagram

```
Rails Partial (hidden)
        |
        v
rewarded-ad controller ──connect──> Native: check subscription + timer
        |                                    |
        v                                    v
   [not subscriber]                    [subscriber]
   reveal promo card                   keep hidden
        |
        ├── Subscribe button ──showPaywall──> Native: RevenueCat PaywallVC
        |                                          |
        |                                     reply(success)
        |
        └── Watch Ad button ──showAd──> Native: AdMob RewardedAd
                                              |
                                         reward earned?
                                         /          \
                                       yes           no
                                        |             |
                                 set 2hr timer    reply(fail)
                                 post notification
                                 reply(success)
```

### Message Flow Reference

| Event | Direction | Component | Payload | Response |
|-------|-----------|-----------|---------|----------|
| `connect` | Web -> Native | `rewarded-ad` | `{ workoutAppId }` | `{ isSubscriber }` |
| `showPaywall` | Web -> Native | `purchase` | `{}` | `{ success, message }` |
| `restorePurchases` | Web -> Native | `purchase` | `{}` | `{ success, message }` |
| `showAd` | Web -> Native | `rewarded-ad` | `{ workoutAppId }` | `{ success, message }` |
| Paywall complete | Native -> Native | `purchase` | -- | Triggers `reply` to web |
| Ad earned reward | Native -> Native | `rewarded-ad` | -- | Sets timer, posts notification |
| Ad dismissed early | Native -> Native | `rewarded-ad` | -- | Triggers `reply(fail)` to web |

> **Note:** This guide covers iOS only. For Android bridge component patterns, see `references/bridge-components.md`.

---

## 2. Web Side -- Rails + Stimulus

### Purchase Bridge Controller

Handles subscription paywall presentation and purchase restoration. Both actions delegate to native SDKs and disable the button during the round-trip.

```javascript
// app/javascript/controllers/purchase_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "purchase"
  static targets = ["paywallButton", "restoreButton"]

  showPaywall() {
    console.log("Requesting paywall from native app...")

    if (this.hasPaywallButtonTarget) {
      this.paywallButtonTarget.disabled = true
    }

    this.send("showPaywall", {}, (message) => {
      if (this.hasPaywallButtonTarget) {
        this.paywallButtonTarget.disabled = false
      }

      if (message.data) {
        const { success, message: responseMessage } = message.data

        if (success) {
          console.log("Paywall displayed successfully")
        } else {
          console.error("Failed to show paywall:", responseMessage)
        }
      }
    })
  }

  restorePurchases() {
    console.log("Requesting purchase restore from native app...")

    if (this.hasRestoreButtonTarget) {
      this.restoreButtonTarget.disabled = true
    }

    this.send("restorePurchases", {}, (message) => {
      if (this.hasRestoreButtonTarget) {
        this.restoreButtonTarget.disabled = false
      }

      if (message.data) {
        const { success, message: responseMessage } = message.data

        if (success) {
          console.log("Restore completed:", responseMessage)
        } else {
          console.error("Restore failed:", responseMessage)
        }
      }
    })
  }
}
```

### Rewarded Ad Bridge Controller

On `connect`, queries native for subscription status and reveals the promo card for non-subscribers. The `showAd` action triggers native ad presentation.

The `workoutAppId` value is your app's context identifier -- pass whatever ID your native side needs to select the correct ad unit or track the context.

```javascript
// app/javascript/controllers/rewarded_ad_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "rewarded-ad"
  static targets = ["widget", "adButton", "adButtonText"]
  static values = {
    workoutAppId: Number
  }

  connect() {
    super.connect()

    // Ask native app for subscription status
    this.send("connect", { workoutAppId: this.workoutAppIdValue }, (message) => {
      if (message.data) {
        const { isSubscriber } = message.data

        if (!isSubscriber) {
          // User is not a subscriber, reveal the widget
          this.element.style.display = "block"
        } else {
          this.element.style.display = "none"
        }
      }
    })
  }

  showAd() {
    if (this.hasAdButtonTarget) {
      this.adButtonTarget.disabled = true

      if (this.hasAdButtonTextTarget) {
        this.adButtonTextTarget.textContent = "Loading..."
      }
    }

    this.send("showAd", { workoutAppId: this.workoutAppIdValue }, (message) => {
      if (this.hasAdButtonTarget) {
        this.adButtonTarget.disabled = false

        if (this.hasAdButtonTextTarget) {
          this.adButtonTextTarget.textContent = "Watch Ad for Free Session"
        }
      }

      if (message.data) {
        const { success, message: responseMessage } = message.data

        if (success) {
          console.log("Rewarded ad completed successfully")
        } else {
          console.error("Failed to show rewarded ad:", responseMessage)
        }
      }
    })
  }
}
```

### Promo Card Partial

Hidden by default with `style="display:none;"`. The `rewarded-ad` controller reveals it on `connect` if the user is not a subscriber. Both bridge controllers are attached to the same root element.

```erb
<!-- app/views/shared/_ad_free_sales_widget.html.erb -->
<div data-controller="rewarded-ad purchase"
     data-rewarded-ad-workout-app-id-value="<%= workout_app.id %>"
     style="display:none;"
     class="block relative rounded-xl overflow-hidden shadow-lg bg-white">

  <div class="p-6">
    <div class="mb-3">
      <h3 class="text-gray-900 text-xl font-normal mb-1">Train Ad-Free.</h3>
      <h3 class="text-indigo-600 text-xl font-bold">Go Premium.</h3>
    </div>

    <p class="text-gray-600 text-sm mb-6">
      Remove ads and enjoy uninterrupted training sessions!
    </p>

    <div class="space-y-3">
      <!-- Subscribe Button -->
      <div data-controller="purchase">
        <button data-action="purchase#showPaywall"
                data-purchase-target="paywallButton"
                class="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold
                       py-3 px-4 rounded-lg transition-colors duration-200 uppercase text-sm">
          Subscribe to Go Ad-Free
        </button>
      </div>

      <!-- Watch Ad Button -->
      <button data-action="rewarded-ad#showAd"
              data-rewarded-ad-target="adButton"
              class="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold
                     py-3 px-4 rounded-lg transition-colors duration-200 uppercase text-sm">
        <span data-rewarded-ad-target="adButtonText">Watch Ad for Free Session</span>
      </button>
    </div>
  </div>
</div>
```

### Controller Registration

Register both controllers in your Stimulus application entry point:

```javascript
// app/javascript/controllers/index.js
import PurchaseController from "./purchase_controller"
application.register("purchase", PurchaseController)

import RewardedAdController from "./rewarded_ad_controller"
application.register("rewarded-ad", RewardedAdController)
```

### Feed Integration

Inject the promo card into a content feed. This example inserts it after the first item:

```erb
<% @training_programs.each_with_index do |training_program, index| %>
  <!-- ... your card markup ... -->

  <% if index == 0 %>
    <%= render "shared/ad_free_sales_widget", workout_app: @workout_app %>
  <% end %>
<% end %>
```

---

## 3. Native Side -- iOS Swift

### PurchaseComponent (RevenueCat)

Presents RevenueCat's `PaywallViewController` when the web side sends `showPaywall`, and restores purchases on `restorePurchases`. Replies to the web side with success/failure after each operation.

```swift
// Components/PurchaseComponent.swift
import HotwireNative
import RevenueCat
import RevenueCatUI
import UIKit

final class PurchaseComponent: BridgeComponent {
    override class var name: String { "purchase" }

    override func onReceive(message: Message) {
        switch message.event {
        case "showPaywall":
            handleShowPaywall(message: message)
        case "restorePurchases":
            handleRestorePurchases(message: message)
        default:
            break
        }
    }

    private func handleShowPaywall(message: Message) {
        guard let viewController = delegate.destination as? UIViewController else {
            reply(to: "showPaywall", with: ["success": false, "message": "No view controller"])
            return
        }

        let paywallVC = PaywallViewController()
        paywallVC.delegate = self
        let nav = UINavigationController(rootViewController: paywallVC)
        viewController.present(nav, animated: true)
    }

    private func handleRestorePurchases(message: Message) {
        Purchases.shared.restorePurchases { customerInfo, error in
            if let error = error {
                self.reply(to: "restorePurchases", with: [
                    "success": false,
                    "message": error.localizedDescription
                ])
            } else {
                let isActive = customerInfo?.entitlements["premium"]?.isActive == true
                self.reply(to: "restorePurchases", with: [
                    "success": true,
                    "message": isActive ? "Premium restored!" : "No active subscription found."
                ])
            }
        }
    }
}

extension PurchaseComponent: PaywallViewControllerDelegate {
    func paywallViewController(_ controller: PaywallViewController,
                               didFinishPurchasingWith customerInfo: CustomerInfo) {
        reply(to: "showPaywall", with: ["success": true])
    }
}
```

### RewardedAdComponent (AdMob + Subscription Check)

Handles the `connect` event by checking subscription state and ad-free timer, preloads ads for non-subscribers, and presents rewarded video on `showAd`.

```swift
// Components/RewardedAdComponent.swift
import HotwireNative
import UIKit
import GoogleMobileAds

final class RewardedAdComponent: BridgeComponent {
    override class var name: String { "rewarded-ad" }

    // Constants
    fileprivate static let adFreeRewardDuration: TimeInterval = 7200 // 2 hours

    // Ad state
    fileprivate var rewardedAd: RewardedAd?
    private var isLoadingAd = false
    fileprivate var pendingShowAdMessage: Message?
    fileprivate var userEarnedReward = false
    private lazy var adContentDelegate = AdContentDelegate(component: self)

    override func onReceive(message: Message) {
        switch message.event {
        case "connect":
            handleConnect(message: message)
        case "showAd":
            handleShowAd(message: message)
        default:
            break
        }
    }

    // MARK: - Connect Event

    private func handleConnect(message: Message) {
        // Check both permanent subscription and temporary ad-free timer
        let isPremium = PaywallManager.shared.isPremiumUser()
        let isAdFree = UserDefaults.standard.isRemoveAds
        let isSubscriber = isPremium || isAdFree

        reply(to: message.event, with: ConnectResponseData(isSubscriber: isSubscriber))

        // Preload a rewarded ad for faster presentation when user taps
        if !isSubscriber {
            preloadRewardedAd()
        }
    }

    // MARK: - Show Ad Event

    private func handleShowAd(message: Message) {
        // Re-check in case user subscribed since connect
        let isPremium = PaywallManager.shared.isPremiumUser()
        let isAdFree = UserDefaults.standard.isRemoveAds

        if isPremium || isAdFree {
            reply(to: message.event, with: ResponseData(
                success: false,
                message: "User already has ad-free access"
            ))
            return
        }

        // Validate ad unit ID from xcconfig
        let adUnitId = RuntimeConfiguration.AdMob.rewardedAdUnitId
        guard !adUnitId.isEmpty else {
            reply(to: message.event, with: ResponseData(
                success: false,
                message: "Rewarded ad unit ID not configured"
            ))
            return
        }

        guard let viewController = self.viewController,
              viewController.presentedViewController == nil else {
            reply(to: message.event, with: ResponseData(
                success: false,
                message: "Another modal is currently presenting"
            ))
            return
        }

        // If ad is preloaded, present immediately. Otherwise load on demand.
        if let loadedAd = rewardedAd {
            presentRewardedAd(loadedAd, from: viewController)
        } else {
            pendingShowAdMessage = message
            loadRewardedAd(from: viewController)
        }
    }

    // MARK: - Ad Loading

    fileprivate func preloadRewardedAd() {
        let adUnitId = RuntimeConfiguration.AdMob.rewardedAdUnitId
        guard !adUnitId.isEmpty, rewardedAd == nil, !isLoadingAd else { return }

        isLoadingAd = true

        Task { @MainActor in
            RewardedAd.load(with: adUnitId, request: Request()) { [weak self] ad, error in
                guard let self = self else { return }
                self.isLoadingAd = false

                if let ad = ad {
                    self.rewardedAd = ad
                    ad.fullScreenContentDelegate = self.adContentDelegate
                }
            }
        }
    }

    private func loadRewardedAd(from viewController: UIViewController) {
        let adUnitId = RuntimeConfiguration.AdMob.rewardedAdUnitId
        guard !adUnitId.isEmpty, !isLoadingAd else { return }

        isLoadingAd = true

        Task { @MainActor in
            RewardedAd.load(with: adUnitId, request: Request()) { [weak self] ad, error in
                guard let self = self else { return }
                self.isLoadingAd = false

                if let error = error {
                    self.reply(to: "showAd", with: ResponseData(
                        success: false,
                        message: "Failed to load ad: \(error.localizedDescription)"
                    ))
                    self.pendingShowAdMessage = nil
                    return
                }

                if let ad = ad {
                    self.rewardedAd = ad
                    ad.fullScreenContentDelegate = self.adContentDelegate
                    self.presentRewardedAd(ad, from: viewController)
                }
            }
        }
    }

    // MARK: - Ad Presentation

    private func presentRewardedAd(_ ad: RewardedAd, from viewController: UIViewController) {
        userEarnedReward = false

        Task { @MainActor in
            ad.present(from: viewController, userDidEarnRewardHandler: { [weak self] in
                self?.userEarnedReward = true
            })
        }
    }
}
```

### AdContentDelegate (Reward Verification + Lifecycle)

A separate `FullScreenContentDelegate` that tracks whether the user earned the reward (watched enough of the ad) versus simply dismissing it. On dismissal, grants the ad-free timer only if the reward was earned, posts a notification for immediate UI updates, and preloads the next ad.

```swift
// Components/RewardedAdComponent.swift (continued)
private class AdContentDelegate: NSObject, FullScreenContentDelegate {
    weak var component: RewardedAdComponent?

    init(component: RewardedAdComponent) {
        self.component = component
    }

    func ad(_ ad: any FullScreenPresentingAd,
            didFailToPresentFullScreenContentWithError error: Error) {
        guard let component = component else { return }

        component.rewardedAd = nil
        component.reply(to: "showAd", with: RewardedAdComponent.ResponseData(
            success: false,
            message: "Failed to present ad: \(error.localizedDescription)"
        ))
        component.pendingShowAdMessage = nil
    }

    func adDidDismissFullScreenContent(_ ad: any FullScreenPresentingAd) {
        guard let component = component else { return }

        component.rewardedAd = nil

        if component.userEarnedReward {
            // Grant 2-hour ad-free period
            let expiryDate = Date().addingTimeInterval(RewardedAdComponent.adFreeRewardDuration)
            UserDefaults.rewardedAdFreeUntil = expiryDate

            // Notify TabBarController to remove banners immediately
            NotificationCenter.default.post(name: .subscriptionStatusChanged, object: nil)

            component.reply(to: "showAd", with: RewardedAdComponent.ResponseData(
                success: true,
                message: "Enjoy 2 hours of ad-free experience!"
            ))
        } else {
            // User dismissed before earning reward
            component.reply(to: "showAd", with: RewardedAdComponent.ResponseData(
                success: false,
                message: "Ad dismissed without earning reward"
            ))
        }

        component.pendingShowAdMessage = nil
        component.preloadRewardedAd() // Preload next ad
    }
}
```

### Component Registration

Register both components during app initialization alongside your other bridge components:

```swift
// AppDelegate.swift or SceneDelegate.swift
Hotwire.registerBridgeComponents([
    PurchaseComponent.self,
    RewardedAdComponent.self,
    // ... your other bridge components
])
```

---

## 4. Ad-Free Timer and Subscription State

The ad-free system combines two independent signals into a single source of truth: a permanent RevenueCat subscription and a temporary 2-hour timer granted by watching a rewarded ad.

### UserDefaults Extension

```swift
// Extensions/UserDefaults+Rewards.swift
extension UserDefaults {
    private static let rewardedAdFreeUntilKey = "REWARDED_AD_FREE_UNTIL"

    static var rewardedAdFreeUntil: Date? {
        get { UserDefaults.standard.object(forKey: rewardedAdFreeUntilKey) as? Date }
        set { UserDefaults.standard.setValue(newValue, forKey: rewardedAdFreeUntilKey) }
    }
}
```

### isRemoveAds -- Single Source of Truth

Combines the permanent subscription flag with the temporary timer. Any code that checks ad-free status uses this single property.

```swift
// Extensions/UserDefaults+Ads.swift
extension UserDefaults {
    var isRemoveAds: Bool {
        // Check permanent ad removal flag (set when subscription is active)
        if bool(forKey: "REMOVE_ADS") { return true }

        // Check temporary rewarded ad-free timer
        if let expiry = UserDefaults.rewardedAdFreeUntil {
            return Date() < expiry
        }

        return false
    }
}
```

### Notification-Driven UI Updates

When a rewarded ad is completed or a subscription is purchased, a `.subscriptionStatusChanged` notification fires. The `TabBarController` (or any observer) listens and removes ad banners from the view hierarchy immediately -- no page reload required.

```swift
// Notifications/SubscriptionNotifications.swift
extension Foundation.Notification.Name {
    static let subscriptionStatusChanged = Foundation.Notification.Name("SubscriptionStatusChanged")
}
```

```swift
// Controllers/TabBarController.swift (relevant excerpt)
private func setupSubscriptionObserver() {
    NotificationCenter.default.addObserver(
        self,
        selector: #selector(subscriptionStatusChanged),
        name: .subscriptionStatusChanged,
        object: nil
    )
}

@objc private func subscriptionStatusChanged() {
    if UserDefaults.standard.isRemoveAds {
        // Remove all existing banner ads from the view hierarchy
        removeBannerAds()
    }
}
```

---

## 5. Configuration (xcconfig to Runtime)

Ad unit IDs flow from build configuration files through Info.plist into Swift at runtime. This avoids hard-coded IDs and supports multiple app targets (e.g., different sports) from a single codebase.

### Base Config.xcconfig

Shared values across all targets. The `ADMOB_REWARDED_ID` is left empty here and overridden per target.

```xcconfig
// Config.xcconfig -- base configuration, shared across all targets
DISPLAY_ADS = true
ADMOB_APP_ID = ca-app-pub-XXXXXXXXXXXXXXXX~XXXXXXXXXX

// Override per target in target-specific .xcconfig files
ADMOB_REWARDED_ID =
```

### Target-Specific Override

Each app target includes the base config and overrides with its own ad unit IDs.

```xcconfig
// Targets/MyApp.xcconfig
#include "Config.xcconfig"

APP_NAME = My App
BUNDLE_IDENTIFIER = com.example.myapp
APP_ID = 123

// AdMob -- production ad unit IDs for this target
ADMOB_APP_ID = ca-app-pub-XXXXXXXXXXXXXXXX~XXXXXXXXXX
ADMOB_BANNER_ID = ca-app-pub-XXXXXXXXXXXXXXXX/XXXXXXXXXX
ADMOB_REWARDED_ID = ca-app-pub-XXXXXXXXXXXXXXXX/XXXXXXXXXX
```

### Info.plist Interpolation

Xcode replaces `$(VARIABLE)` tokens with xcconfig values at build time.

```xml
<!-- Info.plist (relevant entries) -->
<key>ADMOB_REWARDED_ID</key>
<string>$(ADMOB_REWARDED_ID)</string>
```

### RuntimeConfiguration Struct

Provides typed Swift access to build-time configuration values read from Info.plist.

```swift
// Configuration/RuntimeConfiguration.swift
struct RuntimeConfiguration {
    struct AdMob {
        static var rewardedAdUnitId: String {
            Config.shared.adMobRewardedID ?? ""
        }
        static var bannerAdUnitId: String {
            Config.shared.adMobBannerID ?? ""
        }
    }
}
```

```swift
// Configuration/Config.swift
class Config {
    static let shared = Config()

    var adMobRewardedID: String? = try? ConfigurationReader.value(for: "ADMOB_REWARDED_ID")
    var adMobBannerID: String? = try? ConfigurationReader.value(for: "ADMOB_BANNER_ID")
}
```

The `ConfigurationReader` reads values from `Bundle.main.infoDictionary`, which Xcode populates from Info.plist at build time.

---

## 6. Integration Checklist

Follow these steps to add RevenueCat + AdMob monetization to an existing Hotwire Native app:

1. **Add SPM packages** -- `RevenueCatUI` and `GoogleMobileAds` via Swift Package Manager
2. **Configure RevenueCat** -- Call `Purchases.configure(withAPIKey:)` in `AppDelegate`
3. **Create RevenueCat entitlement** -- Set up a `premium` entitlement and paywall in the RevenueCat dashboard
4. **Create AdMob ad units** -- Create a rewarded ad unit in the AdMob console
5. **Set up xcconfig** -- Add `ADMOB_REWARDED_ID` to base `Config.xcconfig` (empty) and override in target-specific xcconfig
6. **Update Info.plist** -- Add `ADMOB_REWARDED_ID` entry with `$(ADMOB_REWARDED_ID)` value
7. **Add Swift components** -- Create `PurchaseComponent.swift` and `RewardedAdComponent.swift`
8. **Register bridge components** -- Add both to `Hotwire.registerBridgeComponents([])`
9. **Deploy Rails changes** -- Add Stimulus controllers and promo card partial, then deploy
10. **Submit iOS update** -- Native changes require App Store review; Rails changes deploy instantly

> **Deployment note:** The Rails-side Stimulus controllers and partial can be deployed independently. They remain inert in web browsers (bridge components only activate inside a Hotwire Native shell). Native changes require an App Store submission.

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Preload ads on `connect` | Eliminates 2-5 second loading delay when user taps "Watch Ad" |
| Track earned vs. dismissed | `userDidEarnRewardHandler` fires only if user watched enough; `adDidDismissFullScreenContent` fires on any close. Reward only granted on earn. |
| Notification-driven banner removal | `.subscriptionStatusChanged` lets any observer (TabBarController, other screens) react immediately without page reloads |
| xcconfig inheritance | Avoids hard-coded ad unit IDs; each target gets correct IDs automatically at build time |
| Hidden partial with `display:none` | Progressive enhancement -- partial is inert in web browsers, revealed only inside native shell by bridge component |
