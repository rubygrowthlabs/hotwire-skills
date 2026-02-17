# rgl-rails

Rails 8 + Full Hotwire Stack skills plugin for Claude Code by [Ruby Growth Labs](https://rubygrowthlabs.com).

## What's Included

7 domain-scoped skills covering the complete Rails + Hotwire stack:

| Skill | Description |
|-------|-------------|
| **rails-architecture** | DHH-style architecture, fat models, thin controllers, REST mapping, Minitest |
| **turbo-navigation** | Turbo Drive caching, Turbo Frames for tabs, pagination, lazy loading, faceted search |
| **turbo-streams** | Real-time updates, broadcasting, custom actions, Turbo 8 morphing, optimistic UI |
| **stimulus-controllers** | Lifecycle, values, targets, outlets, action parameters, production controllers |
| **forms-validation** | Inline editing, modal forms, typeahead, external form controls, submission lifecycle |
| **hotwire-native** | Turbo iOS/Android, path configuration, Strada bridge components, native auth |
| **frontend-craft** | CSS @layer, loading indicators, view transitions, optimistic morphing, frame spinners |

## Installation

```bash
claude plugin install rgl-rails@rubygrowthlabs
```

Or clone directly:

```bash
git clone https://github.com/rubygrowthlabs/rgl-rails.git ~/.claude/skills/rgl-rails
```

## Commands

- `/rails:review` - Review code against DHH/Rails conventions
- `/rails:analyze` - Scan codebase for simplification opportunities
- `/rails:simplify [goal]` - Plan incremental refactoring toward vanilla Rails

## Trigger Phrases

Skills auto-activate on relevant keywords:

- **rails-architecture**: "controller", "concern", "vanilla rails", "dhh style", "minitest", "fixture"
- **turbo-navigation**: "turbo frame", "lazy load", "pagination", "turbo drive", "cache", "tabs"
- **turbo-streams**: "turbo stream", "broadcast", "real-time", "morph", "custom action"
- **stimulus-controllers**: "stimulus", "controller", "values", "targets", "outlets"
- **forms-validation**: "form", "validation", "inline edit", "typeahead", "modal form"
- **hotwire-native**: "turbo native", "ios", "android", "strada", "bridge component", "mobile"
- **frontend-craft**: "css", "loading", "spinner", "view transition", "animation"

## License

MIT
