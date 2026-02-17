---
title: State as Records
category: architecture
updated: 2026-02-17
---

# State as Records

Track state transitions with dedicated models instead of boolean columns. `Card.joins(:closure)` is more powerful than `Card.where(closed: true)`.

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [From Boolean to Record](#from-boolean-to-record)
  - [State Record Models](#state-record-models)
  - [Predicate Methods from Record Presence](#predicate-methods-from-record-presence)
  - [Querying with Joins](#querying-with-joins)
  - [Multiple State Dimensions](#multiple-state-dimensions)
- [Pattern Card](#pattern-card)

## Overview

Boolean columns are the most common way to track state in Rails applications, and they are almost always the wrong choice. A `closed` boolean tells you that a card is closed, but not:

- **When** it was closed
- **Who** closed it
- **Why** it was closed (optional notes)
- The full **audit trail** of open/close transitions

A dedicated state record (`Closure`, `Publication`, `Archival`) gives you all of this for free. The record's `created_at` is the timestamp. The `belongs_to :user` is the actor. You can add any additional context as columns on the state record.

**From Fizzy (37signals Campfire):**

```ruby
# Fizzy never uses boolean columns for state.
# Instead, every state is a dedicated model:
has_one :closure       # Card is closed when closure exists
has_one :goldness      # Card is golden when goldness exists
has_one :not_now       # Card is postponed when not_now exists
```

**The query advantage:** Boolean columns require `WHERE` clauses. State records enable `JOIN`-based queries that are more composable and can include data from the state record itself (like who performed the action and when).

```ruby
# Boolean approach: limited
Card.where(closed: true)

# Record approach: composable, includes actor and timestamp
Card.joins(:closure)
Card.joins(:closure).where(closures: { user: admin })
Card.joins(:closure).order(closures: { created_at: :desc })
```

## Implementation

### From Boolean to Record

**Migration: Remove boolean, add state record**

```ruby
# Step 1: Create the state record table
class CreateClosures < ActiveRecord::Migration[8.0]
  def change
    create_table :closures do |t|
      t.references :card, null: false, foreign_key: true, index: { unique: true }
      t.references :user, null: false, foreign_key: true
      t.text :reason
      t.timestamps
    end
  end
end

# Step 2: Migrate existing data (if applicable)
class MigrateClosedCards < ActiveRecord::Migration[8.0]
  def up
    Card.where(closed: true).find_each do |card|
      Closure.create!(
        card: card,
        user_id: card.closed_by_id || card.creator_id,
        created_at: card.closed_at || card.updated_at
      )
    end
  end
end

# Step 3: Remove boolean columns (separate deploy)
class RemoveClosedFromCards < ActiveRecord::Migration[8.0]
  def change
    remove_column :cards, :closed, :boolean
    remove_column :cards, :closed_at, :datetime
    remove_column :cards, :closed_by_id, :bigint
  end
end
```

### State Record Models

State records are simple models. Their existence IS the state.

```ruby
# app/models/closure.rb
class Closure < ApplicationRecord
  belongs_to :card
  belongs_to :user

  validates :card_id, uniqueness: true

  after_create_commit -> { card.broadcast_replace_later }
  after_destroy_commit -> { card.broadcast_replace_later }
end
```

```ruby
# app/models/publication.rb
class Publication < ApplicationRecord
  belongs_to :post
  belongs_to :user

  validates :post_id, uniqueness: true

  after_create_commit :notify_subscribers_later

  private
    def notify_subscribers_later
      NotifySubscribersJob.perform_later(post)
    end
end
```

### Predicate Methods from Record Presence

The predicate method checks whether the state record exists, not a boolean column:

```ruby
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  # Predicate derived from record presence
  def closed?
    closure.present?
  end

  def open?
    !closed?
  end

  def close(user: Current.user)
    create_closure!(user: user) unless closed?
  end

  def reopen(user: Current.user)
    closure&.destroy! if closed?
  end
end
```

### Querying with Joins

State records unlock powerful query composition:

```ruby
# Basic state queries
Card.closed                    # Card.joins(:closure)
Card.open                      # Card.where.missing(:closure)

# Who closed it?
Card.joins(:closure).where(closures: { user: admin })

# Recently closed
Card.joins(:closure).order(closures: { created_at: :desc })

# Closed this week
Card.joins(:closure).where(closures: { created_at: 1.week.ago.. })

# Closed by a specific team
Card.joins(closure: :user).where(users: { team: engineering })

# Combine state dimensions
Card.joins(:closure).where.missing(:archival)  # Closed but not archived
Card.where.missing(:closure, :archival)        # Open and not archived
```

### Multiple State Dimensions

A model can have multiple independent state dimensions, each tracked by its own record:

```ruby
class Card < ApplicationRecord
  include Closeable     # has_one :closure
  include Archivable    # has_one :archival
  include Publishable   # has_one :publication
  include Postponable   # has_one :not_now (Fizzy's name for postponement)

  # Compound state queries
  scope :active, -> { where.missing(:closure, :archival) }
  scope :stale, -> { active.where(updated_at: ...30.days.ago) }
end
```

Each state dimension is independent. A card can be:
- Open and published
- Closed and archived
- Published but postponed

This is far more flexible than a single `status` enum, which forces mutually exclusive states.

## Pattern Card

### GOOD: Closure Model for Card State

```ruby
# app/models/closure.rb
class Closure < ApplicationRecord
  belongs_to :card
  belongs_to :user
end

# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def closed?
    closure.present?
  end

  def close(user: Current.user)
    create_closure!(user: user) unless closed?
  end

  def reopen(user: Current.user)
    closure&.destroy! if closed?
  end
end

# Usage in controller
class Cards::ClosuresController < ApplicationController
  def create
    @card.close(user: Current.user)
    redirect_to @card.board
  end

  def destroy
    @card.reopen(user: Current.user)
    redirect_to @card.board
  end
end

# Querying
Card.closed                                              # All closed cards
Card.open                                                # All open cards
Card.joins(:closure).where(closures: { user: admin })    # Closed by admin
Card.joins(:closure).order(closures: { created_at: :desc }) # Recently closed first
```

**Why this is good:**
- `Closure` record captures who closed it and when, automatically
- `closed?` predicate is derived from record presence
- Scopes use joins, which are composable with other scopes
- Controller maps to REST: `POST /cards/:id/closure` and `DELETE /cards/:id/closure`
- Adding a `reason` field later is a single column on `closures`, not a migration on `cards`

### BAD: Boolean Column for Card State

```ruby
# app/models/card.rb
class Card < ApplicationRecord
  scope :closed, -> { where(closed: true) }
  scope :open, -> { where(closed: false) }

  def close(user:)
    update!(
      closed: true,
      closed_at: Time.current,
      closed_by_id: user.id
    )
  end

  def reopen(user:)
    update!(
      closed: false,
      closed_at: nil,
      closed_by_id: nil
    )
  end
end

# Controller with custom actions
class CardsController < ApplicationController
  def close
    @card.close(user: Current.user)
    redirect_to @card.board
  end

  def reopen
    @card.reopen(user: Current.user)
    redirect_to @card.board
  end
end

# Routes with custom actions
resources :cards do
  post :close
  post :reopen
end
```

**Why this is bad:**
- Boolean gives you closed/not-closed, nothing more
- `closed_at` and `closed_by_id` are denormalized data on the cards table
- Reopening destroys the history of who closed it and when
- No audit trail of open/close transitions
- Custom controller actions (`close`, `reopen`) instead of REST resources
- Cannot query "cards closed by admin this week" without additional columns
- Every new state dimension (archived, published) adds more boolean columns to cards
