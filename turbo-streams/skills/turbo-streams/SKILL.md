---
name: turbo-streams
description: >-
  Build real-time features with Turbo Streams: inline stream tags, custom stream actions,
  broadcasting over WebSocket, Turbo 8 morphing, optimistic UI, and list animations with
  view transitions. Use when adding real-time updates, broadcasting, or morphing.
  Cross-references: stimulus-controllers for client-side orchestration, hotwire-native for mobile broadcasting.
allowed-tools: Read, Grep, Glob, Task
---

# Turbo Streams

Build real-time, push-driven features using Turbo Streams with Rails.

## Core Workflow

1. **Identify what needs real-time updates.** Determine which parts of the page should change in response to user actions or background events. Look for lists, counters, notifications, status indicators, and collaborative editing surfaces.
2. **Choose delivery method.** Use HTTP response streams (`.turbo_stream.erb` templates) for request/response cycles. Use WebSocket broadcasts (`broadcasts_to`, `turbo_stream_from`) for pushing updates to other users or tabs without a request.
3. **Select the stream action.** Pick the most precise action: `append`/`prepend` for adding to lists, `replace` for swapping an entire element, `update` for changing inner content only, `remove` for deletion, `before`/`after` for positional insertion, `morph` for preserving DOM state during complex updates.
4. **Implement with proper targets and partials.** Use `dom_id` for target IDs. Extract reusable partials. Render the same partial for both initial page load and stream updates to keep markup in sync.
5. **Test both HTTP and WebSocket paths.** Verify that `.turbo_stream` format responses work in controller tests. Use system tests with multiple sessions to confirm broadcasts reach other clients.

## Guardrails

- Prefer `turbo_stream` response format (`.turbo_stream.erb` templates) over inline stream tags when the update originates from a form submission or controller action.
- Scope broadcasts by tenant using `[Current.account, resource]` to prevent data leaking across accounts.
- Use `morph` for complex updates that need to preserve DOM state (form inputs, scroll position, focus).
- Always provide an HTML fallback for non-Turbo clients by using `respond_to` with both `format.turbo_stream` and `format.html`.
- Keep stream payloads small. Send targeted partial updates, not entire page sections.
- Avoid embedding `<script>` tags inside stream templates. Use custom stream actions instead.

## Load References Selectively

Open only the file needed for the current request.

- Inline stream tags in ERB templates: `references/inline-stream-tags.md`
- Custom Turbo Stream actions beyond the 7 built-ins: `references/custom-stream-actions.md`
- Broadcasting with ActionCable/Solid Cable: `references/broadcasting-patterns.md`
- Turbo 8 morphing for page updates: `references/turbo-8-morphing.md`
- Optimistic UI patterns: `references/optimistic-ui.md`
- List animations with View Transitions API: `references/list-animations-view-transitions.md`

Use `references/INDEX.md` for the full catalog with descriptions.

For troubleshooting morph issues, see `examples/morphing-troubleshooting.md`.

### Official Documentation

When you need precise stream action names, element attributes, or API details, read from the `handbook/` directory. Use `handbook/INDEX.md` for the full catalog.

- **Turbo Streams handbook: delivery, actions, targets** -- Read `handbook/turbo-streams.md`
- **Streams API reference (all actions, attributes, events)** -- Read `handbook/turbo-streams-api.md`

## Escalate to Neighbor Skills

- Client-side Stimulus orchestration alongside streams: use `stimulus-controllers`
- Mobile app broadcasting and native bridge concerns: use `hotwire-native`
- Form validation and submit lifecycle: use `forms-validation`
- Navigation, lazy loading, and Turbo Frames: use `turbo-navigation`
