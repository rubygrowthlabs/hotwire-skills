# Turbo Navigation

Navigation patterns with Turbo Drive and Turbo Frames.

## Hotwire Focus

- Turbo Drive
- Turbo Frames

## Articles in this skill

- [Turbo Drive Cache Lifecycle](turbo-drive-cache-lifecycle.md) - How Turbo Drive intercepts links, caches page snapshots, and restores them for instant preview navigations. Covers Cache-Control headers, snapshot cleanup, and stale data prevention.
- [Turbo Frames Tabbed Navigation](turbo-frames-tabbed-navigation.md) - Implementing tabbed content sections where each tab loads its content into a shared Turbo Frame without full page navigation. Covers active state management and back button behavior.
- [Turbo Frames Pagination](turbo-frames-pagination.md) - Frame-scoped pagination that updates only the list content without reloading the entire page. Includes infinite scroll alternative using IntersectionObserver.
- [Turbo Frames Lazy Loading](turbo-frames-lazy-loading.md) - Deferred content loading with the `loading: :lazy` attribute on Turbo Frames. Covers placeholder content, lifecycle events, and performance optimization.
- [Turbo Frames Faceted Search](turbo-frames-faceted-search.md) - Building filter and search interfaces where controls update a results frame without page reload. Covers auto-submit, URL parameter preservation, and combined filter logic.
- [Turbo 8 Page Refresh](turbo-8-page-refresh.md) - Using Turbo 8 morphing page refresh to update pages while preserving scroll position and DOM state. Covers `turbo_refreshes_with`, permanent elements, and when to choose morph vs replace.
