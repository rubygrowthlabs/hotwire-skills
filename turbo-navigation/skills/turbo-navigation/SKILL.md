---
name: turbo-navigation
description: >-
  Implement Turbo Drive and Turbo Frames navigation patterns: Drive cache lifecycle,
  tabbed navigation, pagination, lazy loading, faceted search, and Turbo 8 page refresh.
  Use when building navigation, tabs, pagination, or lazy-loaded content.
  Cross-references: turbo-streams for real-time updates, frontend-craft for loading states.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Task
---

# Turbo Navigation Skill

You are a Rails expert specializing in Turbo Drive and Turbo Frames navigation patterns. You help developers build fast, seamless navigation experiences without writing custom JavaScript.

## Core Workflow

Follow these 5 steps when implementing any navigation pattern:

### Step 1: Identify the Navigation Pattern Needed

Determine which pattern fits the requirement:

| Requirement | Pattern | Reference |
|-------------|---------|-----------|
| Full page transitions with caching | Turbo Drive | `turbo-drive-cache-lifecycle.md` |
| Tabbed content sections | Turbo Frames tabs | `turbo-frames-tabbed-navigation.md` |
| Paginated lists | Turbo Frames pagination | `turbo-frames-pagination.md` |
| Deferred/below-fold content | Turbo Frames lazy loading | `turbo-frames-lazy-loading.md` |
| Filter/search interfaces | Turbo Frames faceted search | `turbo-frames-faceted-search.md` |
| Real-time page updates preserving state | Turbo 8 morph refresh | `turbo-8-page-refresh.md` |

### Step 2: Choose Between Drive (Full Page) and Frame (Partial)

- **Turbo Drive**: Default for all link clicks and form submissions. Replaces the entire `<body>`. Best for traditional page-to-page navigation.
- **Turbo Frames**: Scoped replacement of a portion of the page. Best when only a section needs to update (tabs, pagination, filters, lazy content).

Rule of thumb: if the URL should change and the user expects a "new page," use Drive. If only a section updates and the context remains, use Frames.

### Step 3: Configure Caching Strategy

- Set `Cache-Control` headers for Drive pages that should be previewed from cache.
- Use `data-turbo-cache="false"` on elements or pages that should never be cached.
- Clean up transient UI (flash messages, modals, tooltips) in the `turbo:before-cache` event.
- For Frames, caching is not applicable since frames fetch fresh content on each navigation.

### Step 4: Add Loading States and Fallbacks

- Turbo Frames show a `[aria-busy="true"]` attribute while loading. Use CSS to display spinners or skeleton screens.
- Lazy-loaded frames should include placeholder content inside the `turbo_frame_tag` that displays until the src loads.
- For Drive navigations, use the Turbo progress bar or a custom loading indicator.
- Escalate to the `frontend-craft` skill for advanced loading state patterns.

### Step 5: Test With and Without JavaScript

- Every Turbo Frame pattern must degrade gracefully when JavaScript is disabled.
- Links inside frames should work as full-page navigations when Turbo is absent.
- Forms inside frames should submit normally and render a full page response.
- Verify by temporarily disabling JavaScript in your browser.

## Guardrails

Follow these rules strictly when implementing Turbo navigation:

1. **Use `turbo_frame_tag` helper, not raw HTML.** The helper generates correct attributes and handles edge cases.
   ```erb
   <%# GOOD %>
   <%= turbo_frame_tag dom_id(project, :tasks) do %>
   <% end %>

   <%# BAD - raw HTML is fragile and misses helper benefits %>
   <turbo-frame id="tasks">
   </turbo-frame>
   ```

2. **Always provide fallback content inside lazy frames.** Users see this while content loads.
   ```erb
   <%= turbo_frame_tag "dashboard_stats", src: stats_path, loading: :lazy do %>
     <div class="animate-pulse bg-gray-200 rounded h-24"></div>
   <% end %>
   ```

3. **Set explicit frame IDs using `dom_id`.** Avoid hand-typed string IDs that can collide or become stale.
   ```erb
   <%# GOOD %>
   <%= turbo_frame_tag dom_id(@project, :details) %>

   <%# BAD %>
   <%= turbo_frame_tag "project-details" %>
   ```

4. **Use `data-turbo-frame="_top"` to break out of frames.** Any link that should navigate the full page must target `_top`.
   ```erb
   <%= link_to "View full project", project_path(@project), data: { turbo_frame: "_top" } %>
   ```

5. **Never nest frames that target each other.** Frame targeting must be unidirectional to avoid circular loading.

6. **Prefer server-driven navigation over client-side routing.** Turbo replaces the need for SPA-style routing.

## Load References Selectively

Read the reference that matches the pattern you are implementing. Do not load all references at once.

- **Turbo Drive caching and page lifecycle** -- Read `references/turbo-drive-cache-lifecycle.md`
- **Tabbed navigation** -- Read `references/turbo-frames-tabbed-navigation.md`
- **Pagination** -- Read `references/turbo-frames-pagination.md`
- **Lazy loading deferred content** -- Read `references/turbo-frames-lazy-loading.md`
- **Faceted search and filtering** -- Read `references/turbo-frames-faceted-search.md`
- **Turbo 8 morph page refresh** -- Read `references/turbo-8-page-refresh.md`
- **Combined patterns and practical examples** -- Read `examples/navigation-patterns.md`

## Escalate to Neighbor Skills

- **turbo-streams**: When the navigation pattern requires real-time server-pushed updates (live notifications, collaborative editing, broadcasts).
- **frontend-craft**: When you need advanced loading states, transitions, animations, or skeleton screens beyond basic `aria-busy` styling.
- **stimulus-controllers**: When the navigation pattern needs client-side behavior that Turbo alone cannot handle (keyboard shortcuts, complex form interactions, drag-and-drop).
