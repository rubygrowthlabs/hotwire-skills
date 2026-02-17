# Frontend Craft

Polish frontend UX with modern CSS and Hotwire patterns. Use this skill when improving perceived performance, adding animations, structuring CSS, or implementing loading feedback.

## Hotwire Focus

- CSS @layer architecture
- OKLCH color system
- View Transitions API
- Turbo loading indicators
- Optimistic updates with morphing

## References

- [CSS @layer Architecture](css-layer-architecture.md) - Organizing CSS with @layer for cascade control, OKLCH colors for perceptual uniformity, native CSS nesting, :has() pseudo-class for parent selection, container queries for component-level responsiveness, and logical properties. The 37signals approach to ~60 minimal utilities vs Tailwind's hundreds.
- [Loading Indicators](loading-indicators.md) - Providing visual feedback during Turbo Drive navigation and form submissions. Covers progress bar customization, form submission indicators with turbo:submit-start/end, data-turbo-submits-with for button text changes, and the 200ms delay pattern to avoid flash on fast loads.
- [View Transitions API](view-transitions.md) - Smooth page transitions using the browser View Transitions API with Turbo. Covers meta tag configuration, CSS view-transition-name for element persistence across navigations, @view-transition rules, custom transition animations, and fallback for unsupported browsers.
- [Optimistic Updates with Morphing](optimistic-updates-morphing.md) - Show expected state immediately, let Turbo 8 morph correct if needed. Covers Stimulus controllers for optimistic DOM updates, server broadcast reconciliation, @starting-style for enter animations on morphed elements, and color-mix() for hover/active state generation.
- [Frame Loading Spinners](frame-spinners.md) - Loading feedback specific to Turbo Frame loading. Covers CSS-only spinners inside lazy-loaded frame placeholders, turbo:before-fetch-request/response events, skeleton screens with CSS animations, aria-busy for accessibility, and fixed-height placeholders to prevent layout shift.

## Examples

- [UX Feedback Patterns](../examples/ux-feedback-patterns.md) - Complete UX feedback patterns combining multiple techniques: like button with optimistic count, toast notifications with CSS slide-in, page transitions with view-transition-name, form submission with button state change, and lazy-loaded dashboard cards with skeleton placeholders.
