# Turbo Streams References

Build real-time features with Turbo Streams in Rails applications. Use this skill when implementing push-based updates, broadcasting, morphing, or stream-driven UI changes.

## Hotwire Focus

- Turbo Streams
- ActionCable / Solid Cable
- View Transitions API

## References

- [Inline Stream Tags](inline-stream-tags.md) - Use `turbo_stream` helper methods in `.turbo_stream.erb` templates for multi-action HTTP responses. Covers append, prepend, replace, remove, update, before, and after actions.
- [Custom Stream Actions](custom-stream-actions.md) - Extend Turbo with custom actions beyond the 7 built-in ones using `StreamActions.install()` and server-side helpers. Covers console_log, redirect, dispatch_event, and set_cookie patterns.
- [Broadcasting Patterns](broadcasting-patterns.md) - Push real-time updates over WebSocket using `broadcasts_to`, `after_create_commit` callbacks, and `turbo_stream_from` in views. Covers tenant-scoped broadcasting and Solid Cable configuration.
- [Turbo 8 Morphing](turbo-8-morphing.md) - Use DOM morphing to preserve form state, scroll position, and focus during page updates. Covers `turbo_refreshes_with`, permanent elements, and frame morphing.
- [Optimistic UI](optimistic-ui.md) - Show expected results before server confirmation using Stimulus controllers that update the DOM optimistically and handle rollback on failure.
- [List Animations with View Transitions](list-animations-view-transitions.md) - Animate stream insertions and removals using CSS `@starting-style`, the View Transitions API, and `turbo:before-stream-render` event hooks.

## Examples

- [Morphing Troubleshooting](../examples/morphing-troubleshooting.md) - Common morphing problems and their solutions in a problem/solution table with code examples.
