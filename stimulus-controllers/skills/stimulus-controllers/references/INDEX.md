# Stimulus Controllers

Controller design patterns for Stimulus.js in Rails applications.

## Hotwire Focus

- Stimulus.js
- Controller lifecycle
- Values API
- Targets API
- Outlets API
- Action descriptors

## Articles in this skill

- [Lifecycle: connect and disconnect](lifecycle-connect-disconnect.md) - How Stimulus controller lifecycle hooks work: initialize, connect, and disconnect. Covers setup/teardown symmetry, MutationObserver patterns, and re-entry handling during Turbo navigations.
- [Values and valueChanged Callbacks](values-changed-callbacks.md) - Declaring typed, reactive state with the Values API. Covers type coercion, defaults, valueChanged callbacks, and how values survive Turbo cache restores.
- [Targets and Target Callbacks](targets-target-callbacks.md) - Registering DOM element references with the Targets API. Covers target callbacks (targetConnected/targetDisconnected), has*Target checks, and dynamic content handling.
- [Outlets API](outlets-api.md) - Connecting controllers to each other via the Outlets API for typed, declarative inter-controller communication. Covers outlet callbacks, element references, and when to prefer outlets over events.
- [Action Parameters and Keyboard Events](action-parameters-keyboard.md) - Passing typed data from HTML to action handlers with action parameters. Covers keyboard event filters, action options (:prevent, :stop, :self, :once), and descriptor syntax.
- [Production-Ready Controllers](production-controllers.md) - Six battle-tested controller patterns from 37signals: clipboard, auto-click, toggle-class, auto-submit, dialog, and local-time. Each under 50 lines with full code and HTML usage.
- [Core Web Vitals](core-web-vitals.md) - How Stimulus impacts LCP, FID/INP, and CLS. Covers lazy controller loading, async initialization, and performance-conscious patterns.
