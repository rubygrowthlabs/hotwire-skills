---
title: Broadcasting with ActionCable and Solid Cable
date: 2025-02-01
categories:
  - Turbo Streams
tags:
  - turbo-streams
  - actioncable
  - solid-cable
  - broadcasting
  - websocket
  - real-time
description: Push real-time updates over WebSocket using broadcasts_to, model callbacks, and turbo_stream_from in views.
---

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [View Subscription with turbo_stream_from](#view-subscription-with-turbo_stream_from)
  - [Model Broadcasting with broadcasts_to](#model-broadcasting-with-broadcasts_to)
  - [Manual Broadcasting with Callbacks](#manual-broadcasting-with-callbacks)
  - [Scoping Broadcasts by Tenant](#scoping-broadcasts-by-tenant)
  - [Broadcasting from Background Jobs](#broadcasting-from-background-jobs)
  - [Solid Cable Configuration](#solid-cable-configuration)
- [Key Points](#key-points)
- [Pattern Card: Scoped Broadcasts](#pattern-card-scoped-broadcasts)

## Overview

Turbo Streams can be delivered over WebSocket in addition to HTTP responses. This enables pushing real-time updates to all connected clients when data changes on the server -- without any client polling. Rails provides two mechanisms: declarative model-level broadcasting (`broadcasts_to`) and manual broadcasting via callbacks or service calls.

On the client side, `turbo_stream_from` creates an ActionCable subscription that listens for stream updates. On the server side, changes broadcast `<turbo-stream>` elements to all subscribers.

For the WebSocket transport, Rails supports Redis-backed ActionCable (default) and Solid Cable (database-backed, no Redis needed).

## Implementation

### View Subscription with turbo_stream_from

Add `turbo_stream_from` in your view to subscribe to a broadcast channel:

```erb
<%# app/views/posts/show.html.erb %>

<%# Subscribe to broadcasts for this post's comments %>
<%= turbo_stream_from @post, :comments %>

<div id="comments">
  <%= render @post.comments %>
</div>

<%= render "comments/form", post: @post, comment: Comment.new %>
```

The `turbo_stream_from` helper generates a `<turbo-cable-stream-source>` element that opens a WebSocket connection and listens for broadcasts on the specified channel.

Multiple arguments are joined to form the channel name: `turbo_stream_from @post, :comments` subscribes to the signed stream name derived from `[@post, :comments]`.

### Model Broadcasting with broadcasts_to

The simplest approach uses the `broadcasts_to` class method, which automatically sets up `after_create_commit`, `after_update_commit`, and `after_destroy_commit` callbacks:

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post

  broadcasts_to ->(comment) { [comment.post, :comments] },
                inserts_by: :append,
                target: "comments"
end
```

This is equivalent to writing three separate callbacks:

```ruby
after_create_commit  -> { broadcast_append_to(post, :comments, target: "comments") }
after_update_commit  -> { broadcast_replace_to(post, :comments) }
after_destroy_commit -> { broadcast_remove_to(post, :comments) }
```

The partial used for broadcasting defaults to the model's partial (`comments/_comment.html.erb`). Override it with the `partial:` option:

```ruby
broadcasts_to ->(comment) { [comment.post, :comments] },
              inserts_by: :prepend,
              target: "comments",
              partial: "comments/comment_card"
```

### Manual Broadcasting with Callbacks

For more control, use explicit callbacks with broadcast methods:

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :conversation

  after_create_commit :broadcast_new_message
  after_update_commit :broadcast_updated_message
  after_destroy_commit :broadcast_removal

  private

  def broadcast_new_message
    broadcast_append_to(
      [conversation, :messages],
      target: "messages",
      partial: "messages/message",
      locals: { message: self, current_user: nil }
    )
  end

  def broadcast_updated_message
    broadcast_replace_to(
      [conversation, :messages],
      partial: "messages/message",
      locals: { message: self, current_user: nil }
    )
  end

  def broadcast_removal
    broadcast_remove_to([conversation, :messages])
  end
end
```

Use the `_later` variants to broadcast asynchronously via Active Job, reducing response latency:

```ruby
after_create_commit -> {
  broadcast_append_later_to(
    [conversation, :messages],
    target: "messages",
    partial: "messages/message",
    locals: { message: self }
  )
}
```

### Scoping Broadcasts by Tenant

In multi-tenant applications, always scope broadcasts to prevent data leaking across accounts:

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :project
  has_one :account, through: :project

  # Scope broadcast to the account
  broadcasts_to ->(task) { [task.account, task.project, :tasks] },
                inserts_by: :append,
                target: "tasks"
end
```

```erb
<%# app/views/projects/show.html.erb %>
<%= turbo_stream_from Current.account, @project, :tasks %>

<div id="tasks">
  <%= render @project.tasks %>
</div>
```

The stream names are signed with Rails message verifier, so clients cannot forge subscriptions. However, you must ensure the stream name includes the tenant scope so that a user on Account A never receives broadcasts meant for Account B.

For applications using `Current.account`:

```ruby
# app/models/concerns/tenant_broadcastable.rb
module TenantBroadcastable
  extend ActiveSupport::Concern

  included do
    after_create_commit :broadcast_tenant_append
    after_update_commit :broadcast_tenant_replace
    after_destroy_commit :broadcast_tenant_remove
  end

  private

  def broadcast_tenant_append
    broadcast_append_later_to(
      [account, self.class.name.underscore.pluralize],
      target: self.class.name.underscore.pluralize,
      partial: to_partial_path
    )
  end

  def broadcast_tenant_replace
    broadcast_replace_later_to(
      [account, self.class.name.underscore.pluralize],
      partial: to_partial_path
    )
  end

  def broadcast_tenant_remove
    broadcast_remove_to([account, self.class.name.underscore.pluralize])
  end

  def account
    respond_to?(:account) ? super : Current.account
  end
end
```

### Broadcasting from Background Jobs

When a background job finishes processing, broadcast the result:

```ruby
# app/jobs/import_contacts_job.rb
class ImportContactsJob < ApplicationJob
  def perform(import)
    import.process!

    # Broadcast completion to the user who started the import
    Turbo::StreamsChannel.broadcast_replace_to(
      [import.account, import.user, :imports],
      target: dom_id(import),
      partial: "imports/import",
      locals: { import: import }
    )
  end
end
```

Use `Turbo::StreamsChannel.broadcast_*_to` when broadcasting from outside a model (jobs, service objects, rake tasks).

### Solid Cable Configuration

Solid Cable is a database-backed ActionCable adapter that eliminates the need for Redis. It stores messages in a database table and polls for new messages.

Add it to your Gemfile:

```ruby
# Gemfile
gem "solid_cable"
```

Install and configure:

```bash
bin/rails solid_cable:install
```

This generates a migration and updates your cable configuration:

```yaml
# config/cable.yml
development:
  adapter: solid_cable
  connects_to:
    database:
      writing: cable
  polling_interval: 0.1.seconds
  message_retention: 1.day

production:
  adapter: solid_cable
  connects_to:
    database:
      writing: cable
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

Configure the database connection in `config/database.yml`:

```yaml
production:
  primary:
    <<: *default
    database: myapp_production
  cable:
    <<: *default
    database: myapp_cable_production
    migrations_paths: db/cable_migrate
```

Solid Cable advantages over Redis:
- No additional infrastructure to manage
- Messages survive server restarts (within retention period)
- Works with SQLite for smaller deployments
- Same deployment model as the rest of your Rails app

## Key Points

- Use `turbo_stream_from` in views to subscribe to broadcast channels
- Use `broadcasts_to` for declarative model-level broadcasting
- Use `after_*_commit` callbacks with `broadcast_*_to` for manual control
- Use `_later` variants to broadcast asynchronously via Active Job
- Always scope broadcasts by tenant in multi-tenant applications
- Use `Turbo::StreamsChannel.broadcast_*_to` when broadcasting from outside models
- Solid Cable provides a Redis-free alternative using your database

## Pattern Card: Scoped Broadcasts

**When to use**: Push real-time updates to connected clients when server-side data changes, especially in multi-tenant applications.

**GOOD - Tenant-scoped broadcast with proper channel naming**:

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :project
  has_one :account, through: :project

  broadcasts_to ->(task) { [task.account, task.project, :tasks] },
                inserts_by: :append,
                target: "tasks"
end
```

```erb
<%# app/views/projects/show.html.erb %>
<%= turbo_stream_from Current.account, @project, :tasks %>

<div id="tasks">
  <%= render @project.tasks %>
</div>
```

**BAD - Unscoped global broadcast leaking data across tenants**:

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  # No tenant scoping -- all users see all tasks!
  broadcasts_to ->(task) { :all_tasks },
                inserts_by: :append,
                target: "tasks"
end
```

```erb
<%# Any user on any account receives every task update %>
<%= turbo_stream_from :all_tasks %>
```
