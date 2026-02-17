---
name: rails-architecture
description: >-
  Apply DHH-style Rails 8 architecture: fat models, thin controllers, concerns for composition,
  REST resource mapping, Minitest with fixtures, and Rails 8 defaults (Solid Queue/Cache/Cable).
  Use when writing Ruby/Rails code, reviewing architecture, or discussing DHH/37signals patterns.
  Cross-references: turbo-navigation for navigation, forms-validation for forms, hotwire-native for mobile.
allowed-tools: Read, Grep, Glob, Task
---

# Rails Architecture

DHH-style Rails 8 architecture for building ambitious applications with the simplest possible stack.

## Core Philosophy

> "Vanilla Rails is plenty of Rails for the vast majority of web apps." -- DHH

> "The best code is the code you don't write. The second best is the code that's obviously correct."

> "Services are the waiting room for abstractions that haven't emerged yet."

**Guiding principles from 37signals/Basecamp production codebases:**

1. **Rich domain models own business logic.** Models are not data containers. They contain validations, scopes, state transitions, domain rules, and intention-revealing APIs. Compose behavior with concerns.
2. **Controllers are thin HTTP handlers.** Parse params, call a model method, respond. If a controller action exceeds 8-10 lines, logic is leaking out of the model.
3. **REST is the universal interface.** Every feature maps to CRUD on a resource. Custom actions become new resources: `close` becomes `POST /cards/:id/closure`.
4. **Concerns are horizontal composition.** Name them as adjectives (`Closeable`, `Publishable`, `Watchable`). Each concern adds one coherent capability to any model that includes it.
5. **State lives in records, not booleans.** `Card.joins(:closure)` beats `Card.where(closed: true)`. Dedicated state models give you timestamps, actors, audit trails, and query flexibility for free.
6. **Database-backed everything.** Solid Queue, Solid Cache, Solid Cable. No Redis, no external dependencies. The database is the single source of truth.
7. **Minitest with fixtures.** Ship what Rails ships. Fixtures are fast, deterministic, and encourage you to think about your data as a coherent whole.

## Core Workflow

When implementing any Rails feature, follow these five steps:

### Step 1: Map the Feature to REST Resources

Every user-facing action should map to standard CRUD on a resource. If you find yourself adding custom controller actions, you need a new resource.

| User Action | Bad (Custom Action) | Good (REST Resource) |
|-------------|--------------------|--------------------|
| Close a card | `POST /cards/:id/close` | `POST /cards/:id/closure` |
| Archive a message | `POST /messages/:id/archive` | `POST /messages/:id/archival` |
| Pin a comment | `POST /comments/:id/pin` | `POST /comments/:id/pin` (singular resource) |
| Publish a draft | `POST /drafts/:id/publish` | `POST /drafts/:id/publication` |
| Lock a thread | `POST /threads/:id/lock` | `POST /threads/:id/lock` (singular resource) |

### Step 2: Model the Domain with Rich Models and Concerns

Put business logic in the model. Compose capabilities with concerns.

```ruby
class Card < ApplicationRecord
  include Closeable, Publishable, Watchable, Commentable

  belongs_to :board
  belongs_to :creator, default: -> { Current.user }

  scope :chronologically, -> { order(created_at: :asc) }
  scope :active, -> { where.missing(:closure) }

  def transfer_to(board)
    transaction do
      update!(board: board)
      track_event(:transferred)
    end
  end
end
```

### Step 3: Keep Controllers Thin

Controllers handle HTTP concerns only: parse params, call models, respond.

```ruby
class CardsController < ApplicationController
  before_action :set_card, only: %i[ show edit update destroy ]

  def create
    @card = @board.cards.create!(card_params)
    redirect_to @card
  end

  def update
    @card.update!(card_params)
    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to @card }
    end
  end

  private
    def set_card
      @card = Card.find(params[:id])
    end

    def card_params
      params.require(:card).permit(:title, :body)
    end
end
```

### Step 4: Test with Minitest and Fixtures

Use what Rails ships. Fixtures give you a coherent dataset. Minitest is fast and simple.

```ruby
class CardTest < ActiveSupport::TestCase
  test "closing a card creates a closure record" do
    card = cards(:open_card)
    assert_difference "Closure.count", 1 do
      card.close(user: users(:admin))
    end
    assert card.closed?
  end

  test "cannot close an already closed card" do
    card = cards(:closed_card)
    assert_no_difference "Closure.count" do
      card.close(user: users(:admin))
    end
  end
end
```

### Step 5: Review Against DHH Principles

Before shipping, ask:
- Does every controller action map to CRUD on a resource?
- Is business logic in models, not controllers or services?
- Are concerns named as adjectives and focused on one capability?
- Is state tracked with records, not boolean columns?
- Am I using database-backed infrastructure (Solid Queue/Cache/Cable)?
- Are tests using Minitest with fixtures?

## Guardrails

These are hard rules. Violating them requires explicit justification.

| Rule | Rationale |
|------|-----------|
| No service objects for single-model operations | If it operates on one model, it belongs on that model |
| No gems when stdlib or Rails suffices | `devise` -> custom auth; `pundit` -> model methods; `sidekiq` -> Solid Queue |
| Database-backed everything | No Redis for queues, cache, or pub/sub. Solid Queue/Cache/Cable use the database |
| Controllers map to CRUD verbs only | Custom actions mean you are missing a resource. Create a new controller |
| No `app/services/` directory by default | Services are justified only for multi-model orchestration or external API integration |
| Fixtures over factories | Fixtures are fast, deterministic, and encourage coherent test data |
| Minitest over RSpec | Ship what Rails ships. Minitest is simple and sufficient |

## Quick Reference

### Naming Conventions

| Category | Convention | Examples |
|----------|-----------|----------|
| Model verbs | Domain language, imperative | `card.close`, `card.gild`, `board.publish` |
| Predicates | Derived from record presence | `card.closed?`, `card.golden?` |
| Concerns | Adjectives describing capability | `Closeable`, `Publishable`, `Watchable` |
| Controllers | Nouns matching REST resources | `Cards::ClosuresController` |
| Scopes | Business terms, not SQL | `chronologically`, `active`, `unassigned`, `preloaded` |
| Jobs | `_later` enqueues, `_now` executes | `relay_later` / `relay_now` |

### REST Mapping Table

| Feature | Verb | Route | Controller#Action |
|---------|------|-------|-------------------|
| Close card | POST | `/cards/:id/closure` | `Cards::ClosuresController#create` |
| Reopen card | DELETE | `/cards/:id/closure` | `Cards::ClosuresController#destroy` |
| Archive message | POST | `/messages/:id/archival` | `Messages::ArchivalsController#create` |
| Unarchive | DELETE | `/messages/:id/archival` | `Messages::ArchivalsController#destroy` |
| Pin comment | POST | `/comments/:id/pin` | `Comments::PinsController#create` |
| Publish draft | POST | `/drafts/:id/publication` | `Drafts::PublicationsController#create` |

### Ruby Syntax Preferences

```ruby
# Symbol arrays with spaces inside brackets
before_action :set_card, only: %i[ show edit update destroy ]

# Private method indentation (no newline after private)
  private
    def set_card
      @card = Card.find(params[:id])
    end

# Expression-less case for conditionals
case
when params[:before].present?
  messages.page_before(params[:before])
else
  messages.last_page
end

# Bang methods for fail-fast (create! not create)
@card = @board.cards.create!(card_params)

# belongs_to with Current defaults
belongs_to :creator, default: -> { Current.user }
```

## Load References Selectively

Read only the references relevant to the current task. Each is self-contained.

| Reference | When to Read |
|-----------|-------------|
| [fat-models-thin-controllers.md](./references/fat-models-thin-controllers.md) | Deciding where business logic belongs, refactoring services to models |
| [concerns-composition.md](./references/concerns-composition.md) | Adding shared behavior to models, organizing model capabilities |
| [state-as-records.md](./references/state-as-records.md) | Tracking state changes, replacing boolean columns |
| [rest-resource-mapping.md](./references/rest-resource-mapping.md) | Designing routes, converting custom actions to resources |
| [minitest-fixtures.md](./references/minitest-fixtures.md) | Writing tests, setting up test data, choosing test patterns |
| [current-attributes-jobs.md](./references/current-attributes-jobs.md) | Request context, background jobs, Current attributes |
| [rails-8-defaults.md](./references/rails-8-defaults.md) | Infrastructure decisions, Solid Queue/Cache/Cable, deployment |

See also: [examples/before-after.md](./examples/before-after.md) for concrete refactoring walkthroughs.

### Official Documentation

- **Rails 8 CLAUDE.md project template** -- Read `handbook/rails-8-claude-md-template.md`

## Escalate to Neighbor Skills

This skill focuses on server-side Rails architecture. For specialized concerns, hand off to:

| Concern | Skill | When |
|---------|-------|------|
| Turbo Frames, Turbo Streams, page navigation | `turbo-navigation` | Building interactive UI, real-time updates, page transitions |
| Form design, validation UX, error handling | `forms-validation` | Building forms, handling validation errors, inline editing |
| iOS/Android native shells, bridge components | `hotwire-native` | Wrapping Rails app in native mobile shell, platform-specific behavior |
