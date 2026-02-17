---
title: Concerns as Horizontal Composition
category: architecture
updated: 2026-02-17
---

# Concerns as Horizontal Composition

Concerns are the Rails mechanism for composing model behavior horizontally. Name them as adjectives. Each concern adds one focused capability.

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Anatomy of a Concern](#anatomy-of-a-concern)
  - [The included Block](#the-included-block)
  - [class_methods Block](#class_methods-block)
  - [Composing a Model from Concerns](#composing-a-model-from-concerns)
  - [Controller Concerns](#controller-concerns)
  - [Naming Conventions](#naming-conventions)
- [Pattern Card](#pattern-card)

## Overview

In DHH-style Rails, concerns are the primary tool for code organization within models. Instead of extracting behavior into service objects, you extract it into concerns that the model includes.

The key insight: a concern describes a **capability** that a model gains. `Closeable` means the model can be closed and reopened. `Publishable` means it can be published and unpublished. `Watchable` means users can watch it for changes.

This is fundamentally different from using concerns as "code-slicing" (breaking a large file into smaller files for organizational convenience). Each concern should represent a cohesive capability that could theoretically be included by any model that needs it.

**From Fizzy (37signals Campfire codebase):**

```ruby
class Card < ApplicationRecord
  include Closeable, Golden, Postponable, Stallable, Triageable, Watchable
end
```

Each concern adds one capability with its own:
- Associations (`has_one :closure`)
- Scopes (`scope :closed, -> { joins(:closure) }`)
- Instance methods (`def close`, `def reopen`)
- Callbacks (if needed)
- Class methods (if needed)

## Implementation

### Anatomy of a Concern

Every concern follows the same structure:

```ruby
# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
    scope :recently_closed_first, -> { closed.order(closures: { created_at: :desc }) }
  end

  def closed?
    closure.present?
  end

  def open?
    !closed?
  end

  def close(user: Current.user)
    unless closed?
      transaction do
        create_closure!(user: user)
        track_event(:closed, creator: user)
      end
    end
  end

  def reopen(user: Current.user)
    if closed?
      transaction do
        closure.destroy!
        track_event(:reopened, creator: user)
      end
    end
  end
end
```

### The included Block

The `included` block runs in the context of the including class. Use it for:

- **Associations:** `has_one`, `has_many`, `belongs_to`
- **Scopes:** Query methods related to this capability
- **Callbacks:** `after_create_commit`, `before_save`, etc.
- **Validations:** Related to this capability

```ruby
module Publishable
  extend ActiveSupport::Concern

  included do
    has_one :publication, dependent: :destroy

    scope :published, -> { joins(:publication) }
    scope :drafts, -> { where.missing(:publication) }

    after_create_commit :notify_subscribers_later, if: :published?
  end

  def published?
    publication.present?
  end

  def publish(user: Current.user)
    create_publication!(user: user) unless published?
  end

  def unpublish
    publication&.destroy!
  end

  private
    def notify_subscribers_later
      NotifySubscribersJob.perform_later(self)
    end
end
```

### class_methods Block

Use `class_methods` when you need to add class-level methods through the concern:

```ruby
module Searchable
  extend ActiveSupport::Concern

  included do
    scope :matching, ->(query) { where("title ILIKE ?", "%#{query}%") }
  end

  class_methods
    def search(query)
      return all if query.blank?
      matching(query).order(:title)
    end

    def reindex_all
      find_each(&:reindex)
    end
  end

  def reindex
    SearchIndex.update(self)
  end
end
```

### Composing a Model from Concerns

A well-composed model reads like a capability list. Each `include` tells you something the model can do.

```ruby
class Card < ApplicationRecord
  include Closeable      # can be closed/reopened, has closure record
  include Publishable    # can be published/unpublished
  include Watchable      # users can watch for changes
  include Commentable    # has comments
  include Eventable      # tracks events/activity
  include Broadcastable  # broadcasts changes via Turbo Streams

  belongs_to :board
  belongs_to :creator, default: -> { Current.user }

  validates :title, presence: true

  scope :chronologically, -> { order(created_at: :asc) }

  def transfer_to(board)
    transaction do
      update!(board: board)
      track_event(:transferred)
    end
  end
end
```

**File organization:** Concerns live alongside their model. For a `Card` model:

```
app/models/
  card.rb
  card/
    closeable.rb
    publishable.rb
    watchable.rb
    commentable.rb
```

For concerns shared across multiple models, place them in `app/models/concerns/`:

```
app/models/concerns/
  eventable.rb
  broadcastable.rb
  searchable.rb
```

### Controller Concerns

Controllers also benefit from concerns, primarily for shared `before_action` filters and authentication:

```ruby
# app/controllers/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :authenticated?
  end

  private
    def require_authentication
      resume_session || request_authentication
    end

    def resume_session
      Current.session = Session.find_by(id: cookies.signed[:session_id])
    end

    def request_authentication
      redirect_to new_session_path
    end

    def authenticated?
      Current.session.present?
    end
end
```

### Naming Conventions

| Convention | Examples | When to Use |
|-----------|----------|-------------|
| Adjective (`-able`, `-ible`) | `Closeable`, `Publishable`, `Searchable` | Model gains a capability |
| Adjective (other) | `Golden`, `Stallable`, `Triageable` | Model gains a quality or state |
| Verb + `-ing` | `Broadcasting`, `Relaying` | Model performs an ongoing action |
| Noun (capability) | `Authentication`, `Authorization` | Controller concern providing a system-level feature |

**The adjective test:** If you cannot describe the concern as an adjective that makes sense after "This model is...", the concern may not be focused enough.

- "This card is Closeable" -- yes
- "This card is Publishable" -- yes
- "This card is DataProcessor" -- no, too vague

## Pattern Card

### GOOD: Focused Concern with One Capability

```ruby
# app/models/card/watchable.rb
module Card::Watchable
  extend ActiveSupport::Concern

  included do
    has_many :watches, dependent: :destroy
    has_many :watchers, through: :watches, source: :user

    scope :watched_by, ->(user) { joins(:watches).where(watches: { user: user }) }
  end

  def watched_by?(user)
    watchers.include?(user)
  end

  def watch(user: Current.user)
    watches.find_or_create_by!(user: user)
  end

  def unwatch(user: Current.user)
    watches.find_by(user: user)&.destroy
  end

  def toggle_watch(user: Current.user)
    if watched_by?(user)
      unwatch(user: user)
    else
      watch(user: user)
    end
  end
end
```

**Why this is good:**
- One capability: watching/unwatching
- Clear adjective name: `Watchable`
- Self-contained: associations, scopes, and methods all related to watching
- Any model could include this concern and gain watch capability
- Predicate method `watched_by?` is intention-revealing

### BAD: God Concern with Unrelated Methods

```ruby
# app/models/concerns/card_helpers.rb
module CardHelpers
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
    has_many :watches, dependent: :destroy
    has_many :comments, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :watched_by, ->(user) { joins(:watches).where(watches: { user: user }) }
    scope :with_comments, -> { joins(:comments) }

    before_save :set_default_title
    after_create_commit :notify_watchers
  end

  def close(user:)
    create_closure!(user: user)
  end

  def watch(user:)
    watches.create!(user: user)
  end

  def add_comment(body:, user:)
    comments.create!(body: body, user: user)
  end

  def formatted_title
    title.titleize
  end

  private
    def set_default_title
      self.title = "Untitled" if title.blank?
    end

    def notify_watchers
      WatcherNotificationJob.perform_later(self)
    end
end
```

**Why this is bad:**
- Named as a noun (`CardHelpers`), not an adjective
- Mixes three unrelated capabilities: closing, watching, commenting
- `formatted_title` and `set_default_title` are unrelated to any of the above
- Cannot be reused by other models -- it is Card-specific code-slicing
- If another model needs watching but not closing, it cannot include this
- This is "breaking up a file" not "composing behavior"
