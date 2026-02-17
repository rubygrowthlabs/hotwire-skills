---
title: Current Attributes and Job Patterns
category: architecture
updated: 2026-02-17
---

# Current Attributes and Job Patterns

`Current.user` and `Current.account` provide request context without passing arguments through every layer. Jobs are shallow wrappers that delegate to model methods.

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Setting Up Current](#setting-up-current)
  - [Using Current in Models](#using-current-in-models)
  - [Shallow Job Wrappers](#shallow-job-wrappers)
  - [The _later / _now Convention](#the-_later--_now-convention)
  - [Job Error Handling](#job-error-handling)
- [Pattern Card](#pattern-card)

## Overview

`Current` attributes are a Rails mechanism for storing request-scoped data (the current user, account, request ID) without threading it through every method signature. DHH introduced this pattern because the alternative -- passing `user:` as a keyword argument to every model method -- creates noise without improving clarity.

**From DHH:**

> "Current attributes are a carefully considered trade-off. Yes, they are thread-local global state. But request context IS global within a request. Pretending otherwise just creates busywork."

In 37signals codebases, `Current` is used for:
- `Current.user` -- the authenticated user
- `Current.session` -- the active session
- `Current.account` -- the account scope (multi-tenancy)
- `Current.request_id` -- for log correlation

**Jobs follow the "shallow wrapper" pattern.** A job should contain zero business logic. It receives arguments, calls a model method, and returns. The model method does the actual work. This means the same logic can be called synchronously (in tests, console, or sync execution) or asynchronously (via the job), and the behavior is identical.

**The naming convention:** Methods that enqueue jobs use the `_later` suffix. The synchronous counterpart uses `_now`.

```ruby
card.relay_later   # Enqueues Event::RelayJob
card.relay_now     # Executes synchronously
```

## Implementation

### Setting Up Current

Define your `Current` class in `app/models/current.rb`:

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  attribute :user_agent, :ip_address

  delegate :user, :account, to: :session, allow_nil: true

  def session=(session)
    super
    self.user_agent = session&.user_agent
    self.ip_address = session&.ip_address
  end
end
```

Set it from a controller concern:

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :set_current_session
    helper_method :signed_in?
  end

  private
    def set_current_session
      Current.session = Session.find_by(id: cookies.signed[:session_id])
    end

    def signed_in?
      Current.session.present?
    end

    def require_authentication
      redirect_to new_session_path unless signed_in?
    end
end
```

### Using Current in Models

Use `Current` for default values in `belongs_to` and as keyword argument defaults:

```ruby
class Card < ApplicationRecord
  # Default creator to Current.user
  belongs_to :creator, class_name: "User", default: -> { Current.user }

  # Use Current.user as default in method signatures
  def close(user: Current.user)
    create_closure!(user: user)
  end

  def transfer_to(board, user: Current.user)
    transaction do
      update!(board: board)
      track_event(:transferred, creator: user)
    end
  end
end
```

**Important:** Always accept `user:` as a keyword argument with `Current.user` as the default. This allows tests to pass explicit users without depending on `Current` being set:

```ruby
# In tests: explicit user, no Current dependency
card.close(user: users(:admin))

# In production: Current.user is the default
card.close  # uses Current.user automatically
```

### Shallow Job Wrappers

Jobs should be 3-5 lines. They receive arguments, call a model method, and return.

```ruby
# app/jobs/event/relay_job.rb
class Event::RelayJob < ApplicationJob
  def perform(event)
    event.relay_now
  end
end

# app/jobs/notify_subscribers_job.rb
class NotifySubscribersJob < ApplicationJob
  def perform(post)
    post.notify_subscribers_now
  end
end

# app/jobs/card/broadcast_job.rb
class Card::BroadcastJob < ApplicationJob
  def perform(card)
    card.broadcast_update_now
  end
end
```

**The rule:** If your job has more than one method call in `perform`, the logic belongs on the model.

### The _later / _now Convention

This naming convention from Fizzy makes the sync/async boundary explicit:

```ruby
# app/models/concerns/event/relaying.rb
module Event::Relaying
  extend ActiveSupport::Concern

  included do
    after_create_commit :relay_later
  end

  # Enqueue the job (async)
  def relay_later
    Event::RelayJob.perform_later(self)
  end

  # Do the actual work (sync)
  def relay_now
    EventRelayer.new(self).relay
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  include Notifiable

  def notify_subscribers_later
    NotifySubscribersJob.perform_later(self)
  end

  def notify_subscribers_now
    subscribers.find_each do |subscriber|
      PostMailer.new_post(subscriber, self).deliver_later
    end
  end
end
```

This pattern gives you:
- **`_later`** for production use (non-blocking, queued)
- **`_now`** for tests, console debugging, and sync execution
- Clear naming that tells you whether the method blocks

### Job Error Handling

Handle errors in the model method, not the job:

```ruby
# Good: error handling in the model
class Account::Import < ApplicationRecord
  def process_now
    processing!
    import_records
    mark_completed
  rescue StandardError => e
    mark_as_failed(error: e.message)
    raise  # Re-raise so the job framework can retry
  end
end

class Account::ImportJob < ApplicationJob
  retry_on StandardError, wait: :polynomially_longer, attempts: 3

  def perform(import)
    import.process_now
  end
end
```

```ruby
# Bad: error handling split between job and model
class Account::ImportJob < ApplicationJob
  def perform(import)
    import.processing!
    import.import_records
    import.mark_completed
  rescue StandardError => e
    import.mark_as_failed(error: e.message)
    raise
  end
end
```

## Pattern Card

### GOOD: Shallow Job Calling Model Method

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  after_create_commit :notify_subscribers_later

  def notify_subscribers_later
    NotifySubscribersJob.perform_later(self)
  end

  def notify_subscribers_now
    subscribers.find_each do |subscriber|
      PostMailer.new_post(subscriber, self).deliver_later
    end
  end
end

# app/jobs/notify_subscribers_job.rb
class NotifySubscribersJob < ApplicationJob
  def perform(post)
    post.notify_subscribers_now
  end
end
```

**Why this is good:**
- Job is a 3-line wrapper with zero logic
- `_later` / `_now` naming makes sync/async boundary explicit
- `notify_subscribers_now` can be called directly in tests or console
- Business logic (who to notify, how to notify) lives on the model
- Callback triggers async notification after commit
- Easy to test: call `notify_subscribers_now` directly

### BAD: Business Logic Inside the Job

```ruby
# app/jobs/notify_subscribers_job.rb
class NotifySubscribersJob < ApplicationJob
  def perform(post)
    subscribers = post.subscribers.where(notifications_enabled: true)

    subscribers.find_each do |subscriber|
      next if subscriber.muted?(post.author)
      next if subscriber.last_notified_at > 5.minutes.ago

      notification = Notification.create!(
        user: subscriber,
        post: post,
        kind: :new_post
      )

      if subscriber.prefers_email?
        PostMailer.new_post(subscriber, post).deliver_later
      end

      if subscriber.prefers_push?
        PushNotificationService.send(subscriber, notification)
      end

      subscriber.update!(last_notified_at: Time.current)
    end
  end
end
```

**Why this is bad:**
- Job contains business logic (filtering, preferences, notification creation)
- Cannot call this logic synchronously without going through the job
- Impossible to test notification logic without enqueuing a job
- Mixes concerns: subscriber filtering, notification creation, delivery preferences
- When notification rules change, you edit the job instead of the model
- The `Post` model has no idea how its subscribers are notified
