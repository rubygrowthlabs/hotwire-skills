# CLAUDE.md

## Plugin Overview

hotwire-skills is a Claude Code skills plugin covering Rails 8 + the full Hotwire stack (Turbo + Stimulus + Native). It contains 7 domain-scoped skills with ~45 reference articles, slash commands, and the only Hotwire Native coverage in the ecosystem.

## Skill Architecture

Each skill follows a three-tier structure:

1. **SKILL.md** - Main skill definition with YAML frontmatter and trigger phrases
2. **commands/*.md** - Slash commands (only in rails-architecture)
3. **agents/*.md** - Task agent definitions using `model: inherit`

Skills also have two knowledge layers:
- **references/** - Synthesized, opinionated guidance (pattern cards, guardrails)
- **handbook/** - Official documentation (Turbo/Stimulus/Hotwire Native API specs, handbook chapters)

## Directory Layout

```
hotwire-skills/
├── rails-architecture/   # Hub skill with commands and agent
│   └── handbook/          # Rails 8 CLAUDE.md template
├── turbo-navigation/     # Turbo Drive + Frames navigation patterns
│   └── handbook/          # Official Turbo handbook + API refs (10 files)
├── turbo-streams/        # Real-time streaming and broadcasting
│   └── handbook/          # Official Turbo Streams handbook + API ref (2 files)
├── stimulus-controllers/ # Stimulus controller fundamentals
│   └── handbook/          # Official Stimulus handbook + API refs (16 files)
├── forms-validation/     # Form handling with Hotwire
├── hotwire-native/       # iOS + Android + Strada (unique differentiator)
│   └── handbook/          # Turbo Native overview + TurboFailureApp auth (2 files)
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
