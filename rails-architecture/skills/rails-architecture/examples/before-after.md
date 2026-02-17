# Before/After Examples

Concrete refactoring walkthroughs showing the service-to-model transformation. Each example starts with a common over-engineered pattern and ends with idiomatic DHH-style Rails.

## Example 1: CreateOrderService -> Order#process

### Before: Service Object for Single-Model Operation

```ruby
# app/services/create_order_service.rb
class CreateOrderService
  def initialize(customer:, items:, coupon_code: nil)
    @customer = customer
    @items = items
    @coupon_code = coupon_code
  end

  def call
    order = Order.new(customer: @customer, status: :pending)

    @items.each do |item_attrs|
      order.line_items.build(
        product: Product.find(item_attrs[:product_id]),
        quantity: item_attrs[:quantity]
      )
    end

    order.subtotal = order.line_items.sum { |li| li.product.price * li.quantity }

    if @coupon_code.present?
      coupon = Coupon.find_by!(code: @coupon_code)
      order.discount = coupon.calculate_discount(order.subtotal)
      order.coupon = coupon
    end

    order.tax = (order.subtotal - (order.discount || 0)) * order.tax_rate
    order.total = order.subtotal - (order.discount || 0) + order.tax

    order.save!

    OrderMailer.confirmation(order).deliver_later
    InventoryService.new(order).reserve_stock
    AnalyticsService.new.track_order(order)

    order
  end
end

# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  def create
    @order = CreateOrderService.new(
      customer: Current.user,
      items: order_params[:items],
      coupon_code: order_params[:coupon_code]
    ).call

    redirect_to @order
  rescue ActiveRecord::RecordInvalid => e
    @order = e.record
    render :new, status: :unprocessable_entity
  end
end
```

**Problems:**
- Service wraps a single model's creation logic
- Order model is anemic: it cannot create itself properly
- Pricing logic belongs on Order, not in a service
- Three additional services called in sequence (inventory, analytics, mailer)

### After: Rich Order Model

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  include Processable, Discountable, Trackable

  belongs_to :customer, class_name: "User"
  belongs_to :coupon, optional: true
  has_many :line_items, dependent: :destroy

  before_save :calculate_totals

  scope :pending, -> { where(status: :pending) }
  scope :completed, -> { where(status: :completed) }
  scope :chronologically, -> { order(created_at: :asc) }

  def process
    transaction do
      calculate_totals
      save!
      reserve_inventory_later
      track_event(:created)
    end
  end

  private
    def calculate_totals
      self.subtotal = line_items.sum { |li| li.product.price * li.quantity }
      apply_coupon_discount if coupon.present?
      self.tax = taxable_amount * tax_rate
      self.total = taxable_amount + tax
    end

    def taxable_amount
      subtotal - (discount || 0)
    end

    def apply_coupon_discount
      self.discount = coupon.calculate_discount(subtotal)
    end
end

# app/models/order/processable.rb
module Order::Processable
  extend ActiveSupport::Concern

  included do
    after_create_commit :send_confirmation_later
  end

  private
    def send_confirmation_later
      OrderMailer.confirmation(self).deliver_later
    end

    def reserve_inventory_later
      Order::ReserveInventoryJob.perform_later(self)
    end
end

# app/controllers/orders_controller.rb
class OrdersController < ApplicationController
  def create
    @order = Current.user.orders.build(order_params)
    @order.process

    redirect_to @order
  rescue ActiveRecord::RecordInvalid
    render :new, status: :unprocessable_entity
  end
end
```

**What changed:**
- `CreateOrderService` deleted entirely (0 LOC)
- Pricing logic moved to `Order#calculate_totals` (where it belongs)
- Mailer triggered by `after_create_commit` callback in `Processable` concern
- Inventory reservation delegated to a job via `_later` pattern
- Event tracking handled by `Trackable` concern
- Controller is 6 lines

---

## Example 2: UserAuthenticationService -> User#authenticate

### Before: Service Object for Authentication

```ruby
# app/services/user_authentication_service.rb
class UserAuthenticationService
  def initialize(email:, password:)
    @email = email
    @password = password
  end

  def authenticate
    user = User.find_by(email: @email)
    return failure("User not found") unless user
    return failure("Account locked") if user.locked?
    return failure("Invalid password") unless user.valid_password?(@password)

    user.update!(
      last_sign_in_at: Time.current,
      sign_in_count: user.sign_in_count + 1,
      failed_attempts: 0
    )

    session = Session.create!(user: user, ip_address: Current.ip_address)
    success(user: user, session: session)
  end

  private

  def success(data)
    OpenStruct.new(success?: true, **data)
  end

  def failure(message)
    OpenStruct.new(success?: false, error: message)
  end
end

# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def create
    result = UserAuthenticationService.new(
      email: params[:email],
      password: params[:password]
    ).authenticate

    if result.success?
      cookies.signed[:session_id] = result.session.id
      redirect_to root_path
    else
      flash.now[:alert] = result.error
      render :new, status: :unprocessable_entity
    end
  end
end
```

**Problems:**
- Service wraps User authentication logic that belongs on User
- Custom result objects (OpenStruct) add unnecessary indirection
- Sign-in tracking logic scattered across service
- User model cannot authenticate itself

### After: Rich User Model with Rails 8 Authentication

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password
  has_many :sessions, dependent: :destroy

  normalizes :email, with: ->(email) { email.strip.downcase }

  def authenticate_and_sign_in(password)
    if authenticate(password)
      return nil if locked?
      record_sign_in
      sessions.create!(ip_address: Current.ip_address, user_agent: Current.user_agent)
    end
  end

  def locked?
    failed_attempts >= 5 && locked_at > 30.minutes.ago
  end

  def lock!
    update!(locked_at: Time.current)
  end

  private
    def record_sign_in
      update!(
        last_sign_in_at: Time.current,
        sign_in_count: sign_in_count + 1,
        failed_attempts: 0
      )
    end
end

# app/controllers/sessions_controller.rb
class SessionsController < ApplicationController
  def create
    user = User.authenticate_by(email: params[:email], password: params[:password])

    if session = user&.authenticate_and_sign_in(params[:password])
      start_new_session_for(user)
      redirect_to root_path
    else
      redirect_to new_session_path, alert: "Invalid email or password"
    end
  end
end
```

**What changed:**
- `UserAuthenticationService` deleted entirely
- Authentication logic lives on `User` where it belongs
- `User#authenticate_and_sign_in` is intention-revealing
- No custom result objects -- returns session or nil
- Uses Rails 8 `has_secure_password` and `authenticate_by`
- Controller is 7 lines
- `locked?` is a predicate on User, not a check in a service

---

## Example 3: NotificationService -> Notifiable Concern

### Before: Service Object for Notifications

```ruby
# app/services/notification_service.rb
class NotificationService
  def initialize(record, event:, recipients: nil)
    @record = record
    @event = event
    @recipients = recipients || default_recipients
  end

  def notify
    @recipients.each do |recipient|
      next if recipient == Current.user
      next unless recipient.notifications_enabled?

      notification = Notification.create!(
        user: recipient,
        notifiable: @record,
        event: @event
      )

      deliver_notification(notification, recipient)
    end
  end

  private

  def default_recipients
    case @record
    when Card
      @record.watchers
    when Comment
      @record.card.watchers
    when Board
      @record.members
    else
      []
    end
  end

  def deliver_notification(notification, recipient)
    if recipient.prefers_email?
      NotificationMailer.notify(notification).deliver_later
    end
    if recipient.prefers_push?
      PushNotificationJob.perform_later(notification)
    end
  end
end

# Called from multiple controllers
class CommentsController < ApplicationController
  def create
    @comment = @card.comments.create!(comment_params)
    NotificationService.new(@comment, event: :new_comment).notify
    redirect_to @card
  end
end
```

**Problems:**
- Service handles notifications for all model types with a case statement
- Each model type has different recipient logic mixed into one service
- Controllers must remember to call the service after every create
- Notification preferences checked in the service, not on the user

### After: Notifiable Concern on Each Model

```ruby
# app/models/concerns/notifiable.rb
module Notifiable
  extend ActiveSupport::Concern

  included do
    has_many :notifications, as: :notifiable, dependent: :destroy
    after_create_commit :notify_watchers_later
  end

  def notify_watchers_later
    NotifyWatchersJob.perform_later(self)
  end

  def notify_watchers_now
    notification_recipients.each do |recipient|
      next if recipient == Current.user

      recipient.notifications.create!(
        notifiable: self,
        event: notification_event
      )
    end
  end

  # Override in including model to customize recipients
  def notification_recipients
    respond_to?(:watchers) ? watchers : []
  end

  # Override in including model to customize event name
  def notification_event
    :created
  end
end

# app/models/comment.rb
class Comment < ApplicationRecord
  include Notifiable

  belongs_to :card
  belongs_to :creator, default: -> { Current.user }

  def notification_recipients
    card.watchers
  end

  def notification_event
    :new_comment
  end
end

# app/models/notification.rb
class Notification < ApplicationRecord
  include Deliverable

  belongs_to :user
  belongs_to :notifiable, polymorphic: true

  after_create_commit :deliver_later
end

# app/models/notification/deliverable.rb
module Notification::Deliverable
  extend ActiveSupport::Concern

  def deliver_later
    Notification::DeliverJob.perform_later(self)
  end

  def deliver_now
    NotificationMailer.notify(self).deliver_later if user.prefers_email?
    PushNotificationJob.perform_later(self) if user.prefers_push?
  end
end

# app/jobs/notify_watchers_job.rb
class NotifyWatchersJob < ApplicationJob
  def perform(record)
    record.notify_watchers_now
  end
end

# app/controllers/comments_controller.rb
class CommentsController < ApplicationController
  def create
    @comment = @card.comments.create!(comment_params)
    redirect_to @card
  end
end
```

**What changed:**
- `NotificationService` deleted entirely
- `Notifiable` concern auto-triggers notifications via `after_create_commit`
- Each model defines its own `notification_recipients` and `notification_event`
- Delivery preferences live on `User` (via `prefers_email?`, `prefers_push?`)
- Notification delivery logic lives on `Notification::Deliverable` concern
- Controller does not need to know about notifications at all
- Adding notifications to a new model: just `include Notifiable`

---

## Key Takeaways

1. **Delete the service, enrich the model.** Every service refactoring starts by moving logic onto the model that owns the data.
2. **Concerns compose behavior.** `Notifiable`, `Processable`, `Trackable` are reusable capabilities, not code-sliced files.
3. **Callbacks handle side effects.** `after_create_commit` triggers notifications, broadcasts, and job enqueuing without cluttering the controller.
4. **The `_later` / `_now` convention** keeps sync and async paths clear and testable.
5. **Controllers get shorter with each refactoring.** The goal is 3-6 lines per action.
