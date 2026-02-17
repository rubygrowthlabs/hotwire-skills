---
title: Rails 8 Defaults
category: infrastructure
updated: 2026-02-17
---

# Rails 8 Defaults

Database-backed everything. Solid Queue, Solid Cache, Solid Cable, Kamal, Propshaft. No external dependencies required.

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Solid Queue (Background Jobs)](#solid-queue-background-jobs)
  - [Solid Cache (HTTP and Fragment Caching)](#solid-cache-http-and-fragment-caching)
  - [Solid Cable (WebSockets / Action Cable)](#solid-cable-websockets--action-cable)
  - [Propshaft (Asset Pipeline)](#propshaft-asset-pipeline)
  - [Kamal (Deployment)](#kamal-deployment)
  - [Authentication Generator](#authentication-generator)
  - [Migration from Redis/Sidekiq](#migration-from-redissidekiq)
- [Pattern Card](#pattern-card)

## Overview

Rails 8 shipped with a philosophy: **the database is enough.** You do not need Redis for queues, caching, or pub/sub. You do not need Sidekiq for background jobs. You do not need a separate process for WebSocket connections. The database handles all of it.

This is the "Solid" trifecta:
- **Solid Queue** -- Background jobs backed by the database (replaces Sidekiq/Redis)
- **Solid Cache** -- Cache store backed by the database (replaces Redis/Memcached)
- **Solid Cable** -- Action Cable adapter backed by the database (replaces Redis pub/sub)

**Why this matters:**

1. **Fewer moving parts.** One database instead of database + Redis + Sidekiq process.
2. **Simpler deployment.** Kamal deploys your app and database. No Redis to provision or monitor.
3. **Simpler development.** `rails server` starts everything. No Redis, no Sidekiq process.
4. **Transactional integrity.** Jobs enqueued in `after_commit` callbacks are guaranteed to see committed data.
5. **Cost reduction.** No Redis hosting fees. No Sidekiq Pro license.

**From DHH:**

> "Most apps don't need Redis. The database is right there. It already handles ACID transactions, replication, and failover. Why add another piece of infrastructure?"

## Implementation

### Solid Queue (Background Jobs)

Solid Queue stores jobs in database tables. It uses `FOR UPDATE SKIP LOCKED` for high-performance polling with no duplicates.

**Setup (default in Rails 8 new apps):**

```ruby
# config/queue.yml
default: &default
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: "*"
      threads: 3
      processes: 1
      polling_interval: 0.1

development:
  <<: *default

production:
  <<: *default
  workers:
    - queues: "*"
      threads: 5
      processes: 2
      polling_interval: 0.1
```

```ruby
# config/application.rb
config.active_job.queue_adapter = :solid_queue
```

```ruby
# config/solid_queue.yml is auto-loaded
# Run migrations:
# bin/rails solid_queue:install:migrations
# bin/rails db:migrate
```

**Usage is standard Active Job:**

```ruby
class NotifySubscribersJob < ApplicationJob
  queue_as :default

  def perform(post)
    post.notify_subscribers_now
  end
end

# Enqueue
NotifySubscribersJob.perform_later(post)
```

**Recurring jobs (replaces cron/whenever):**

```ruby
# config/recurring.yml
production:
  cleanup_old_sessions:
    class: CleanupSessionsJob
    schedule: every day at 3am
  send_daily_digest:
    class: DailyDigestJob
    schedule: every day at 8am
```

### Solid Cache (HTTP and Fragment Caching)

Solid Cache stores cache entries in the database. It is designed for large caches (gigabytes) with automatic expiration.

**Setup:**

```ruby
# config/cache.yml
default: &default
  store_options:
    max_age: <%= 60.days.to_i %>
    max_size: <%= 256.megabytes %>
    namespace: <%= Rails.env %>

production:
  database: cache
  <<: *default
```

```ruby
# config/environments/production.rb
config.cache_store = :solid_cache_store
```

**Usage is standard Rails caching:**

```ruby
# Fragment caching in views
<% cache @card do %>
  <%= render @card %>
<% end %>

# Low-level caching in models
def expensive_calculation
  Rails.cache.fetch("card/#{id}/stats", expires_in: 1.hour) do
    calculate_stats
  end
end

# HTTP caching in controllers
def show
  @card = Card.find(params[:id])
  fresh_when @card
end
```

### Solid Cable (WebSockets / Action Cable)

Solid Cable provides a database-backed pub/sub adapter for Action Cable. It uses database polling instead of Redis pub/sub.

**Setup:**

```ruby
# config/cable.yml
development:
  adapter: solid_cable
  polling_interval: 0.1

production:
  adapter: solid_cable
  polling_interval: 0.1
```

**Usage is standard Action Cable:**

```ruby
# Broadcasting with Turbo Streams
class Card < ApplicationRecord
  after_update_commit -> { broadcast_replace_later_to board }
  after_destroy_commit -> { broadcast_remove_to board }
end

# Direct broadcasts
Turbo::StreamsChannel.broadcast_replace_later_to(
  board,
  target: dom_id(card),
  partial: "cards/card",
  locals: { card: card }
)
```

### Propshaft (Asset Pipeline)

Propshaft replaces Sprockets as the default asset pipeline. It is simpler and faster because it does not compile or transpile assets -- it only digests and serves them.

```ruby
# Gemfile (default in Rails 8)
gem "propshaft"

# Use import maps for JavaScript (no Node.js needed)
gem "importmap-rails"

# app/assets/config/manifest.js is not needed with Propshaft
# Assets in app/assets/ are served automatically
```

**CSS with Propshaft:**

```css
/* app/assets/stylesheets/application.css */
@layer base, components, utilities;

@layer base {
  :root {
    --color-primary: oklch(0.6 0.2 250);
  }
}

@layer components {
  .card {
    border: 1px solid var(--color-primary);
  }
}
```

### Kamal (Deployment)

Kamal deploys your app to any server with Docker and SSH. No PaaS needed.

```yaml
# config/deploy.yml
service: myapp
image: myapp/web

servers:
  web:
    hosts:
      - 192.168.1.1
  job:
    hosts:
      - 192.168.1.1
    cmd: bin/jobs

registry:
  username: myuser
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
```

### Authentication Generator

Rails 8 includes a built-in authentication generator. No Devise needed.

```bash
bin/rails generate authentication
```

This generates:
- `User` model with `has_secure_password`
- `Session` model for session tracking
- `SessionsController` for login/logout
- `Current` class with session/user delegation
- `Authentication` concern for controllers
- Database migrations for users and sessions

```ruby
# Generated: app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :session
  delegate :user, to: :session, allow_nil: true
end

# Generated: app/models/session.rb
class Session < ApplicationRecord
  belongs_to :user
end

# Generated: app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def create
    user = User.authenticate_by(
      email_address: params[:email_address],
      password: params[:password]
    )

    if user
      start_new_session_for(user)
      redirect_to root_path
    else
      redirect_to new_session_path, alert: "Invalid credentials"
    end
  end
end
```

### Migration from Redis/Sidekiq

If you have an existing app using Redis and Sidekiq, the migration path is incremental:

**Step 1: Add Solid Queue alongside Sidekiq**

```ruby
# Gemfile
gem "solid_queue"
# Keep sidekiq for now

# config/application.rb
# Keep Sidekiq as default, route specific jobs to Solid Queue
```

**Step 2: Migrate jobs one at a time**

```ruby
class LowPriorityJob < ApplicationJob
  self.queue_adapter = :solid_queue  # This job uses Solid Queue
  # ...
end
```

**Step 3: Switch default adapter**

```ruby
config.active_job.queue_adapter = :solid_queue
```

**Step 4: Remove Redis/Sidekiq**

```ruby
# Gemfile: remove sidekiq, redis gems
# config: remove redis.yml, sidekiq.yml
# Infrastructure: decommission Redis
```

## Pattern Card

### GOOD: Database-Backed Everything

```ruby
# Gemfile
gem "rails", "~> 8.0"
gem "solid_queue"
gem "solid_cache"
gem "solid_cable"
gem "propshaft"
gem "importmap-rails"
gem "kamal"

# No redis gem
# No sidekiq gem
# No memcached gem

# config/application.rb
config.active_job.queue_adapter = :solid_queue
config.cache_store = :solid_cache_store

# config/cable.yml
production:
  adapter: solid_cable

# Infrastructure: just the app server and database
# No Redis to provision, monitor, or pay for
```

**Why this is good:**
- One database handles jobs, cache, and pub/sub
- `rails server` starts everything in development
- Fewer moving parts in production
- No Redis hosting costs
- Transactional integrity: jobs see committed data
- Standard Rails APIs: Active Job, Rails.cache, Action Cable

### BAD: External Service Dependencies

```ruby
# Gemfile
gem "rails", "~> 8.0"
gem "sidekiq", "~> 7.0"
gem "sidekiq-pro"       # $180/year license
gem "redis", "~> 5.0"
gem "hiredis"           # Redis C binding
gem "connection_pool"   # Redis connection pooling
gem "memcachier"        # Memcached client

# config/application.rb
config.active_job.queue_adapter = :sidekiq
config.cache_store = :mem_cache_store

# config/cable.yml
production:
  adapter: redis
  url: <%= ENV["REDIS_URL"] %>

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV["REDIS_URL"] }
end

# Infrastructure requirements:
# - Redis instance (for Sidekiq + Action Cable)
# - Memcached instance (for caching)
# - Sidekiq process (separate from web)
# - Redis monitoring/alerting
# - Redis backup strategy
```

**Why this is bad:**
- Three external services (Redis, Memcached, Sidekiq) when the database suffices
- Redis is a single point of failure for jobs AND real-time features
- Extra hosting costs: Redis, Memcached, Sidekiq Pro license
- Extra operational burden: monitoring, backups, upgrades for each service
- Development setup requires Redis and Sidekiq running locally
- `Procfile.dev` needed to start multiple processes
- When Redis goes down, jobs stop AND Action Cable disconnects
