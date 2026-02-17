---
title: Fat Models, Thin Controllers
category: architecture
updated: 2026-02-17
---

# Fat Models, Thin Controllers

Rich domain models over service objects. Business logic belongs on the model that owns the data.

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Rich Model Methods](#rich-model-methods)
  - [Intention-Revealing APIs](#intention-revealing-apis)
  - [When Services ARE Justified](#when-services-are-justified)
  - [Thin Controller Patterns](#thin-controller-patterns)
- [Pattern Card](#pattern-card)

## Overview

The single most important architectural decision in a Rails application is where business logic lives. In DHH-style Rails, the answer is always the same: **on the model**.

Models are not data containers. They are the domain. A `Card` model knows how to close itself, gild itself, transfer itself between boards, and track events. The controller's only job is to receive the HTTP request, call the appropriate model method, and respond.

This is the opposite of the "service object" pattern popular in enterprise-influenced Rails codebases, where models are reduced to schema definitions and all behavior is extracted into `app/services/`. That pattern creates indirection without value: you now have two files to read instead of one, and the model no longer tells you what a Card can do.

**The test:** Open any model file. Can you tell what the domain object is capable of just by reading the class? If the model only has `belongs_to`, `has_many`, and `validates`, it is anemic and the architecture is wrong.

**Key insight from 37signals:** Fizzy (Basecamp's Campfire codebase) has **zero** service objects. Every operation is a method on a model, composed via concerns. Complex multi-step processes use plain Ruby objects with `ActiveModel::Model` or ActiveRecord models with status tracking.

## Implementation

### Rich Model Methods

A rich model exposes domain operations as methods. Each method encapsulates the full behavior, including side effects like event tracking and notifications.

```ruby
class Order < ApplicationRecord
  include Processable, Cancellable, Refundable

  belongs_to :customer
  belongs_to :creator, default: -> { Current.user }
  has_many :line_items, dependent: :destroy

  scope :pending, -> { where(status: :pending) }
  scope :completed, -> { where(status: :completed) }
  scope :chronologically, -> { order(created_at: :asc) }

  def process
    transaction do
      calculate_totals
      apply_discounts
      update!(status: :processing)
      track_event(:processed)
    end
  end

  def complete
    transaction do
      update!(status: :completed, completed_at: Time.current)
      send_confirmation_later
      track_event(:completed)
    end
  end

  private
    def calculate_totals
      self.subtotal = line_items.sum(&:total)
      self.tax = subtotal * tax_rate
      self.total = subtotal + tax
    end

    def apply_discounts
      return unless customer.has_active_discount?
      self.discount = customer.discount_for(subtotal)
      self.total -= discount
    end

    def send_confirmation_later
      OrderMailer.confirmation(self).deliver_later
    end
end
```

### Intention-Revealing APIs

Model methods should read like natural language. The caller should not need to understand the internals.

```ruby
# Good: intention-revealing
card.close(user: Current.user)
card.transfer_to(other_board)
order.process
user.authenticate(password)
account.provision_for(user)

# Bad: implementation-revealing
card.update!(closed: true, closed_at: Time.current, closed_by: user.id)
CardTransferService.new(card, other_board).call
ProcessOrderService.new(order).call
UserAuthenticationService.new(user, password).authenticate
AccountProvisioningService.call(account, user)
```

The model method hides the complexity. The controller just calls `card.close` and does not care whether that creates a `Closure` record, tracks an event, or sends a notification. That is the model's concern.

### When Services ARE Justified

Services are the exception, not the rule. They are justified in exactly three cases:

**1. Multi-model orchestration with no natural owner**

When an operation spans multiple unrelated models and no single model is the obvious owner:

```ruby
# Justified: Signup touches User, Account, Identity, and Subscription
# No single model owns the full signup flow
class Signup
  include ActiveModel::Model

  attr_accessor :email, :name, :plan

  validates :email, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :name, presence: true

  def save
    transaction do
      @account = Account.create!(name: name)
      @user = @account.users.create!(email: email, name: name)
      @subscription = @account.subscriptions.create!(plan: plan)
    end
  end
end
```

**2. External API integration**

When the operation is fundamentally about an external system:

```ruby
# Justified: The core operation is talking to Stripe
class StripeChargeCreator
  def initialize(order)
    @order = order
  end

  def create
    Stripe::Charge.create(
      amount: @order.total_cents,
      currency: "usd",
      customer: @order.customer.stripe_id
    )
  end
end
```

**3. Stateful multi-step processes**

When the process has its own lifecycle and needs status tracking, use an ActiveRecord model:

```ruby
# Justified: Import has its own state (pending, processing, completed, failed)
class Account::Import < ApplicationRecord
  enum :status, %w[ pending processing completed failed ].index_by(&:itself)

  def process(start: nil, callback: nil)
    processing!
    import_records(start: start)
    mark_completed(callback: callback)
  rescue => e
    mark_as_failed
    raise
  end
end
```

### Thin Controller Patterns

Controllers should be so thin that they are almost boring. Here are the three patterns you need:

**Standard CRUD:**

```ruby
class CardsController < ApplicationController
  def create
    @card = @board.cards.create!(card_params)
    redirect_to @card
  end

  def update
    @card.update!(card_params)
    redirect_to @card
  end

  def destroy
    @card.destroy!
    redirect_to @board
  end
end
```

**With Turbo Stream responses:**

```ruby
class Cards::ClosuresController < ApplicationController
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
end
```

**With JSON API:**

```ruby
class Api::CardsController < ApplicationController
  def create
    @card = @board.cards.create!(card_params)
    render json: @card, status: :created
  end

  def update
    @card.update!(card_params)
    render json: @card
  end
end
```

## Pattern Card

### GOOD: Rich Model with Domain Method

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  def process
    transaction do
      calculate_totals
      update!(status: :processing)
      track_event(:processed)
      send_confirmation_later
    end
  end
end

# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  def create
    @order = current_account.orders.create!(order_params)
    @order.process
    redirect_to @order
  end
end
```

**Why this is good:**
- `Order` knows how to process itself
- Controller is 3 lines
- Opening `order.rb` tells you everything an order can do
- One file to read, one concept to understand

### BAD: Service Object Wrapping Single Model

```ruby
# app/services/process_order_service.rb
class ProcessOrderService
  def initialize(order)
    @order = order
  end

  def call
    @order.calculate_totals
    @order.update!(status: :processing)
    EventTracker.track(@order, :processed)
    OrderMailer.confirmation(@order).deliver_later
    @order
  end
end

# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  def create
    @order = current_account.orders.create!(order_params)
    ProcessOrderService.new(@order).call
    redirect_to @order
  end
end
```

**Why this is bad:**
- The service does nothing the model cannot do
- `Order` is now anemic -- it cannot process itself
- You must read two files to understand order processing
- The service is a thin wrapper adding indirection without value
- Opening `order.rb` tells you nothing about what an order can do
