---
title: REST Resource Mapping
category: architecture
updated: 2026-02-17
---

# REST Resource Mapping

Every action in your application maps to CRUD on a resource. If you are adding custom controller actions, you are missing a resource.

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Converting Custom Actions to Resources](#converting-custom-actions-to-resources)
  - [Singular Resources](#singular-resources)
  - [Nested Resources](#nested-resources)
  - [Namespace Organization](#namespace-organization)
  - [Route Design Patterns](#route-design-patterns)
- [Pattern Card](#pattern-card)

## Overview

REST resource mapping is the foundational architectural constraint in DHH-style Rails. The principle is simple: **every user action is CRUD (Create, Read, Update, Delete) on a resource.** If you find yourself adding a custom action to a controller, you have not found the right resource yet.

This constraint has profound architectural benefits:

1. **Predictable controllers.** Every controller has at most 7 actions: `index`, `show`, `new`, `create`, `edit`, `update`, `destroy`. No surprises.
2. **Discoverable routes.** `rails routes` tells the full story. No custom actions to guess.
3. **Consistent patterns.** Every developer on the team writes controllers the same way.
4. **Natural mapping to state records.** "Close a card" becomes "Create a closure." The REST resource IS the state record.

**From DHH's blog post "How DHH Organizes His Rails Controllers":**

> "Anytime I find a controller action that's not one of the standard CRUD operations, I know I'm missing a concept. I need to find that concept and give it a name."

The mapping table:

| User Intent | Wrong (Custom Action) | Right (REST Resource) |
|-------------|----------------------|-----------------------|
| Close a card | `POST /cards/:id/close` | `POST /cards/:id/closure` |
| Reopen a card | `POST /cards/:id/reopen` | `DELETE /cards/:id/closure` |
| Archive a message | `POST /messages/:id/archive` | `POST /messages/:id/archival` |
| Unarchive | `POST /messages/:id/unarchive` | `DELETE /messages/:id/archival` |
| Publish a post | `POST /posts/:id/publish` | `POST /posts/:id/publication` |
| Pin a comment | `POST /comments/:id/pin` | `POST /comments/:id/pin` |
| Lock a thread | `POST /threads/:id/lock` | `POST /threads/:id/lock` |
| Mark as read | `POST /messages/:id/mark_read` | `POST /messages/:id/reading` |
| Star an item | `POST /items/:id/star` | `POST /items/:id/star` |

## Implementation

### Converting Custom Actions to Resources

The conversion process: identify the verb, find the noun, create a controller.

**Step 1: Identify the verb in the custom action**

```ruby
# Custom action: "close"
resources :cards do
  post :close       # The verb is "close"
  post :reopen      # The verb is "reopen" (reverse of close)
end
```

**Step 2: Find the noun (the resource)**

"Close" -> "Closure". The noun form of the action becomes the resource.

Common verb-to-noun mappings:

| Verb | Noun (Resource) |
|------|----------------|
| close | closure |
| archive | archival |
| publish | publication |
| approve | approval |
| lock | lock |
| pin | pin |
| star | star |
| watch | watch |
| subscribe | subscription |
| bookmark | bookmark |
| read / mark read | reading |
| complete | completion |

**Step 3: Create a nested resource controller**

```ruby
# config/routes.rb
resources :cards do
  resource :closure, only: %i[ create destroy ]
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  before_action :set_card

  def create
    @card.close(user: Current.user)
    redirect_to @card.board
  end

  def destroy
    @card.reopen(user: Current.user)
    redirect_to @card.board
  end

  private
    def set_card
      @card = Card.find(params[:card_id])
    end
end
```

### Singular Resources

When a parent has exactly one of something, use `resource` (singular) instead of `resources` (plural):

```ruby
# A card has one closure (not many)
resources :cards do
  resource :closure, only: %i[ create destroy ]
  resource :publication, only: %i[ create destroy ]
  resource :archival, only: %i[ create destroy ]
end
```

This generates:
- `POST /cards/:card_id/closure` -> `Cards::ClosuresController#create`
- `DELETE /cards/:card_id/closure` -> `Cards::ClosuresController#destroy`

No `:id` parameter needed because the resource is singular (one closure per card).

### Nested Resources

Nest resources to express ownership. Keep nesting shallow (max 2 levels):

```ruby
# Two-level nesting (good)
resources :boards do
  resources :cards, only: %i[ index create ]
end

resources :cards do
  resource :closure, only: %i[ create destroy ]
  resources :comments, only: %i[ index create destroy ]
end

# Shallow nesting (preferred for deep hierarchies)
resources :boards, shallow: true do
  resources :cards do
    resources :comments
  end
end
```

**Shallow nesting** generates member routes (`show`, `edit`, `update`, `destroy`) without the parent prefix, since the resource ID is sufficient:

```
# Collection routes include parent
GET    /boards/:board_id/cards          cards#index
POST   /boards/:board_id/cards          cards#create

# Member routes are shallow
GET    /cards/:id                       cards#show
PATCH  /cards/:id                       cards#update
DELETE /cards/:id                       cards#destroy
```

### Namespace Organization

Use namespaces to group related controllers:

```ruby
# Admin namespace
namespace :admin do
  resources :users
  resources :accounts
end

# API namespace with versioning
namespace :api do
  namespace :v1 do
    resources :cards, only: %i[ index show create update destroy ]
  end
end

# Nested state controllers under resource namespace
resources :cards do
  resource :closure, module: :cards, only: %i[ create destroy ]
  resource :goldness, module: :cards, only: %i[ create destroy ]
end
```

### Route Design Patterns

**Pattern: Toggle actions as create/destroy**

Toggling state maps to creating and destroying the state record:

```ruby
resources :cards do
  resource :closure, only: %i[ create destroy ]    # close / reopen
  resource :goldness, only: %i[ create destroy ]   # gild / ungild
  resource :pin, only: %i[ create destroy ]        # pin / unpin
end
```

**Pattern: Show + Update for settings**

When a resource has editable settings:

```ruby
resources :accounts do
  resource :settings, only: %i[ show update ]
end
```

**Pattern: Collection actions as separate controllers**

When you need filtered views of a collection:

```ruby
# Instead of: GET /cards?filter=closed
# Use:
resources :cards           # All cards (or open cards)

namespace :cards do
  resources :closed, only: :index, controller: "closed"
  resources :archived, only: :index, controller: "archived"
end
```

## Pattern Card

### GOOD: Nested Resource Controller

```ruby
# config/routes.rb
resources :cards do
  resource :closure, only: %i[ create destroy ]
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  before_action :set_card

  def create
    @card.close(user: Current.user)

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to @card.board }
    end
  end

  def destroy
    @card.reopen(user: Current.user)

    respond_to do |format|
      format.turbo_stream
      format.html { redirect_to @card.board }
    end
  end

  private
    def set_card
      @card = Card.find(params[:card_id])
    end
end
```

**Why this is good:**
- Standard CRUD actions only (`create` and `destroy`)
- Controller is named after the resource (`ClosuresController`), not the action
- Routes are predictable: `POST /cards/:id/closure`, `DELETE /cards/:id/closure`
- Controller is thin: calls model method, responds
- Maps naturally to state-as-records pattern

### BAD: Custom Action Methods on Parent Controller

```ruby
# config/routes.rb
resources :cards do
  member do
    post :close
    post :reopen
    post :archive
    post :unarchive
    post :publish
    post :unpublish
    post :pin
    post :unpin
  end
end

# app/controllers/cards_controller.rb
class CardsController < ApplicationController
  before_action :set_card

  def show; end
  def edit; end

  def close
    @card.update!(closed: true, closed_at: Time.current, closed_by: Current.user)
    redirect_to @card.board, notice: "Card closed"
  end

  def reopen
    @card.update!(closed: false, closed_at: nil, closed_by: nil)
    redirect_to @card.board, notice: "Card reopened"
  end

  def archive
    @card.update!(archived: true, archived_at: Time.current)
    redirect_to @card.board, notice: "Card archived"
  end

  def unarchive
    @card.update!(archived: false, archived_at: nil)
    redirect_to @card.board, notice: "Card unarchived"
  end

  # ... and so on for publish, pin, etc.
end
```

**Why this is bad:**
- `CardsController` has 12+ actions instead of the standard 7
- Custom routes are unpredictable and undiscoverable
- State managed with boolean columns instead of records
- Each toggle pair (close/reopen, archive/unarchive) duplicates the same pattern
- Controller is fat with business logic (setting timestamps, clearing fields)
- No separation of concerns: one controller handles everything a card can do
- Cannot reuse the close/reopen pattern for other models
