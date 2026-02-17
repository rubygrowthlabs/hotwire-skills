# CLAUDE.md

## Plugin Overview

rgl-rails is a Claude Code skills plugin covering Rails 8 + the full Hotwire stack (Turbo + Stimulus + Native). It contains 7 domain-scoped skills with ~45 reference articles, slash commands, and the only Hotwire Native coverage in the ecosystem.

## Skill Architecture

Each skill follows a three-tier structure:

1. **SKILL.md** - Main skill definition with YAML frontmatter and trigger phrases
2. **commands/*.md** - Slash commands (only in rails-architecture)
3. **agents/*.md** - Task agent definitions using `model: inherit`

## Directory Layout

```
rgl-rails/
├── rails-architecture/   # Hub skill with commands and agent
├── turbo-navigation/     # Turbo Drive + Frames navigation patterns
├── turbo-streams/        # Real-time streaming and broadcasting
├── stimulus-controllers/ # Stimulus controller fundamentals
├── forms-validation/     # Form handling with Hotwire
├── hotwire-native/       # iOS + Android + Strada (unique differentiator)
└── frontend-craft/       # CSS architecture and UX feedback
```

## Cross-Skill Routing

Skills route to each other via "Escalate to Neighbor Skills" sections in each SKILL.md. When a request crosses domains, defer to the appropriate sibling skill.

## Versioning

When releasing changes, increment the version in:
- Root `.claude-plugin/marketplace.json` (top-level `metadata.version` AND each skill's `version`)
- Per-skill `.claude-plugin/plugin.json` (`version`)

## Development

- Each reference article follows: Overview -> Implementation -> Pattern Card (GOOD/BAD)
- SKILL.md files follow: Core Workflow (5 steps) -> Guardrails -> Load References Selectively -> Escalate to Neighbor Skills
- Agent files must use `model: inherit`
