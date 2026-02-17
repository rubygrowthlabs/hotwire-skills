---
title: "Navigation Patterns: Combined Examples"
categories:
  - turbo
  - navigation
  - examples
tags:
  - turbo-frames
  - turbo-drive
  - lazy-loading
  - tabs
  - pagination
  - faceted-search
  - dashboard
description: >-
  Practical examples combining multiple Turbo navigation patterns into
  real-world page architectures. Includes a dashboard with lazy-loaded widgets,
  a search page with faceted filters and paginated results, and a settings
  page with tabbed sections.
---

# Navigation Patterns: Combined Examples

## Table of Contents

- [Example 1: Dashboard with Lazy-Loaded Widgets](#example-1-dashboard-with-lazy-loaded-widgets)
- [Example 2: Search Page with Faceted Filters and Paginated Results](#example-2-search-page-with-faceted-filters-and-paginated-results)
- [Example 3: Settings Page with Tabbed Sections](#example-3-settings-page-with-tabbed-sections)

---

## Example 1: Dashboard with Lazy-Loaded Widgets

A dashboard page that loads instantly with critical content, then lazily loads secondary widgets as the user scrolls. Uses Turbo 8 morph refresh to keep widgets updated in real time.

**Patterns combined:** Lazy loading, Turbo 8 morph refresh, Turbo Drive caching

### Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resource :dashboard, only: [:show] do
    member do
      get :recent_projects
      get :activity_feed
      get :team_stats
      get :upcoming_deadlines
    end
  end
end
```

### Controller

```ruby
# app/controllers/dashboards_controller.rb
class DashboardsController < ApplicationController
  def show
    @greeting = greeting_for(Current.user)
    @notifications_count = Current.user.notifications.unread.count
    @quick_actions = Current.user.pending_approvals.limit(5)
  end

  def recent_projects
    @projects = Current.user.projects
      .includes(:last_activity)
      .order(updated_at: :desc)
      .limit(6)
  end

  def activity_feed
    @activities = Current.user.team.activities
      .includes(:user, :trackable)
      .order(created_at: :desc)
      .limit(15)
  end

  def team_stats
    @stats = TeamStatsQuery.new(Current.user.team).call
  end

  def upcoming_deadlines
    @deadlines = Current.user.assigned_tasks
      .incomplete
      .where(due_date: Date.today..14.days.from_now)
      .order(:due_date)
      .limit(10)
  end

  private

  def greeting_for(user)
    hour = Time.current.hour
    period = case hour
             when 5..11  then "morning"
             when 12..16 then "afternoon"
             else "evening"
             end
    "Good #{period}, #{user.first_name}"
  end
end
```

### Layout Configuration

```erb
<%# app/views/layouts/application.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <%= turbo_refreshes_with method: :morph, scroll: :preserve %>
  <%= yield :head %>
</head>
<body>
  <nav id="main-nav" data-turbo-permanent>
    <%= render "shared/navigation" %>
  </nav>
  <main>
    <%= yield %>
  </main>
</body>
</html>
```

### Dashboard View

```erb
<%# app/views/dashboards/show.html.erb %>
<%= turbo_stream_from "dashboard_#{Current.user.id}" %>

<div class="dashboard">
  <%# Critical content loads with the page -- no lazy loading %>
  <section class="dashboard-header">
    <h1><%= @greeting %></h1>
    <div class="quick-stats">
      <span class="stat">
        <strong><%= @notifications_count %></strong> unread notifications
      </span>
    </div>
  </section>

  <% if @quick_actions.any? %>
    <section class="quick-actions">
      <h2>Pending Approvals</h2>
      <ul class="action-list">
        <% @quick_actions.each do |approval| %>
          <li>
            <%= link_to approval.title, approval_path(approval), data: { turbo_frame: "_top" } %>
            <span class="badge"><%= approval.priority %></span>
          </li>
        <% end %>
      </ul>
    </section>
  <% end %>

  <%# Secondary content loads lazily as user scrolls %>
  <section class="dashboard-widgets">
    <div class="widget-row">
      <%= turbo_frame_tag "recent_projects",
        src: recent_projects_dashboard_path,
        loading: :lazy do %>
        <div class="widget">
          <h3>Recent Projects</h3>
          <div class="widget-skeleton">
            <% 3.times do %>
              <div class="skeleton-card">
                <div class="skeleton-line w-3/4"></div>
                <div class="skeleton-line w-1/2"></div>
              </div>
            <% end %>
          </div>
        </div>
      <% end %>

      <%= turbo_frame_tag "team_stats",
        src: team_stats_dashboard_path,
        loading: :lazy do %>
        <div class="widget">
          <h3>Team Stats</h3>
          <div class="skeleton-block h-48"></div>
        </div>
      <% end %>
    </div>

    <div class="widget-row">
      <%= turbo_frame_tag "activity_feed",
        src: activity_feed_dashboard_path,
        loading: :lazy do %>
        <div class="widget widget--wide">
          <h3>Activity Feed</h3>
          <% 5.times do %>
            <div class="skeleton-activity">
              <div class="skeleton-circle w-8 h-8"></div>
              <div class="skeleton-line flex-1"></div>
            </div>
          <% end %>
        </div>
      <% end %>

      <%= turbo_frame_tag "upcoming_deadlines",
        src: upcoming_deadlines_dashboard_path,
        loading: :lazy do %>
        <div class="widget">
          <h3>Upcoming Deadlines</h3>
          <% 4.times do %>
            <div class="skeleton-line w-full"></div>
          <% end %>
        </div>
      <% end %>
    </div>
  </section>
</div>
```

### Widget Views

```erb
<%# app/views/dashboards/recent_projects.html.erb %>
<%= turbo_frame_tag "recent_projects" do %>
  <div class="widget">
    <h3>Recent Projects</h3>
    <div class="project-cards">
      <% @projects.each do |project| %>
        <div class="project-card" id="<%= dom_id(project) %>">
          <h4><%= link_to project.name, project_path(project), data: { turbo_frame: "_top" } %></h4>
          <p class="text-sm text-gray-500">
            Updated <%= time_ago_in_words(project.updated_at) %> ago
          </p>
          <div class="progress-bar">
            <div class="progress-fill" style="width: <%= project.completion_percentage %>%"></div>
          </div>
        </div>
      <% end %>
    </div>
  </div>
<% end %>
```

```erb
<%# app/views/dashboards/activity_feed.html.erb %>
<%= turbo_frame_tag "activity_feed" do %>
  <div class="widget widget--wide">
    <h3>Activity Feed</h3>
    <ul class="activity-list">
      <% @activities.each do |activity| %>
        <li class="activity-item" id="<%= dom_id(activity) %>">
          <div class="activity-avatar">
            <%= image_tag activity.user.avatar_url, alt: activity.user.name, class: "avatar-sm" %>
          </div>
          <div class="activity-content">
            <p>
              <strong><%= activity.user.name %></strong>
              <%= activity.description %>
            </p>
            <time class="text-sm text-gray-400" datetime="<%= activity.created_at.iso8601 %>">
              <%= time_ago_in_words(activity.created_at) %> ago
            </time>
          </div>
        </li>
      <% end %>
    </ul>
  </div>
<% end %>
```

```erb
<%# app/views/dashboards/team_stats.html.erb %>
<%= turbo_frame_tag "team_stats" do %>
  <div class="widget">
    <h3>Team Stats</h3>
    <dl class="stats-grid">
      <div class="stat-card">
        <dt>Tasks Completed</dt>
        <dd><%= @stats.tasks_completed_this_week %></dd>
      </div>
      <div class="stat-card">
        <dt>Active Projects</dt>
        <dd><%= @stats.active_projects_count %></dd>
      </div>
      <div class="stat-card">
        <dt>Team Members</dt>
        <dd><%= @stats.team_members_count %></dd>
      </div>
      <div class="stat-card">
        <dt>Avg. Completion Time</dt>
        <dd><%= @stats.avg_completion_days %> days</dd>
      </div>
    </dl>
  </div>
<% end %>
```

```erb
<%# app/views/dashboards/upcoming_deadlines.html.erb %>
<%= turbo_frame_tag "upcoming_deadlines" do %>
  <div class="widget">
    <h3>Upcoming Deadlines</h3>
    <% if @deadlines.any? %>
      <ul class="deadline-list">
        <% @deadlines.each do |task| %>
          <li class="deadline-item <%= 'deadline--urgent' if task.due_date <= 3.days.from_now %>">
            <%= link_to task.title, project_task_path(task.project, task), data: { turbo_frame: "_top" } %>
            <time datetime="<%= task.due_date.iso8601 %>">
              <%= task.due_date.strftime("%b %d") %>
            </time>
          </li>
        <% end %>
      </ul>
    <% else %>
      <p class="empty-state">No upcoming deadlines. Nice work!</p>
    <% end %>
  </div>
<% end %>
```

---

## Example 2: Search Page with Faceted Filters and Paginated Results

A product catalog page with a filter sidebar, text search, and paginated results. Filters and search auto-submit to update the results frame.

**Patterns combined:** Faceted search, pagination, Turbo Drive caching

### Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :products, only: [:index, :show]
end
```

### Controller

```ruby
# app/controllers/products_controller.rb
class ProductsController < ApplicationController
  include Pagy::Backend

  def index
    scope = Product.includes(:category, :brand).with_attached_image

    scope = scope.search(params[:q]) if params[:q].present?
    scope = scope.where(category_id: params[:category]) if params[:category].present?
    scope = scope.where(brand_id: params[:brand]) if params[:brand].present?
    scope = scope.where(status: params[:status]) if params[:status].present?
    scope = scope.in_price_range(params[:price_min], params[:price_max])

    scope = apply_sort(scope, params[:sort])

    @pagy, @products = pagy(scope, items: 24)
    @categories = Category.ordered
    @brands = Brand.with_products
    @active_filters_count = count_active_filters
  end

  private

  def apply_sort(scope, sort)
    case sort
    when "price_asc"     then scope.order(price: :asc)
    when "price_desc"    then scope.order(price: :desc)
    when "name"          then scope.order(name: :asc)
    when "newest"        then scope.order(created_at: :desc)
    when "best_selling"  then scope.order(sales_count: :desc)
    else                      scope.order(featured: :desc, created_at: :desc)
    end
  end

  def count_active_filters
    [:q, :category, :brand, :status, :price_min, :price_max].count { |k| params[k].present? }
  end
end
```

### View

```erb
<%# app/views/products/index.html.erb %>
<div class="catalog-page">
  <header class="catalog-header">
    <h1>Products</h1>
  </header>

  <div class="catalog-layout">
    <%# Filter sidebar stays outside the results frame %>
    <aside class="catalog-filters">
      <%= form_with url: products_path, method: :get, class: "filter-form", data: {
        turbo_frame: "products_results",
        controller: "auto-submit",
        auto_submit_delay_value: 300
      } do |f| %>

        <div class="filter-section">
          <h3>Search</h3>
          <%= f.search_field :q,
            value: params[:q],
            placeholder: "Search products...",
            class: "filter-input",
            data: { action: "input->auto-submit#submit" } %>
        </div>

        <div class="filter-section">
          <h3>Category</h3>
          <%= f.select :category,
            options_for_select(@categories.map { |c| [c.name, c.id] }, params[:category]),
            { include_blank: "All categories" },
            class: "filter-select",
            data: { action: "change->auto-submit#submit" } %>
        </div>

        <div class="filter-section">
          <h3>Brand</h3>
          <%= f.select :brand,
            options_for_select(@brands.map { |b| [b.name, b.id] }, params[:brand]),
            { include_blank: "All brands" },
            class: "filter-select",
            data: { action: "change->auto-submit#submit" } %>
        </div>

        <div class="filter-section">
          <h3>Status</h3>
          <% %w[available pre_order out_of_stock].each do |status| %>
            <label class="filter-checkbox">
              <%= check_box_tag "status[]", status,
                params[:status]&.include?(status),
                data: { action: "change->auto-submit#submit" } %>
              <span><%= status.titleize %></span>
            </label>
          <% end %>
        </div>

        <div class="filter-section">
          <h3>Price range</h3>
          <div class="price-range">
            <%= f.number_field :price_min,
              value: params[:price_min],
              placeholder: "Min",
              min: 0,
              class: "filter-input filter-input--small",
              data: { action: "change->auto-submit#submit" } %>
            <span class="price-separator">&ndash;</span>
            <%= f.number_field :price_max,
              value: params[:price_max],
              placeholder: "Max",
              min: 0,
              class: "filter-input filter-input--small",
              data: { action: "change->auto-submit#submit" } %>
          </div>
        </div>

        <div class="filter-actions">
          <%= f.submit "Apply", class: "btn btn-sm btn-primary" %>
          <% if @active_filters_count > 0 %>
            <%= link_to "Clear all (#{@active_filters_count})", products_path, class: "btn btn-sm btn-secondary" %>
          <% end %>
        </div>
      <% end %>
    </aside>

    <%# Results frame includes product grid AND pagination %>
    <main class="catalog-results">
      <%= turbo_frame_tag "products_results" do %>
        <div class="results-toolbar">
          <p class="results-count">
            <%= pluralize(@pagy.count, "product") %> found
          </p>

          <div class="results-sort">
            <%= form_with url: products_path, method: :get, data: {
              turbo_frame: "products_results",
              controller: "auto-submit"
            } do |f| %>
              <%# Carry forward all current filters as hidden fields %>
              <% %i[q category brand price_min price_max].each do |key| %>
                <% if params[key].present? %>
                  <%= f.hidden_field key, value: params[key] %>
                <% end %>
              <% end %>
              <% if params[:status].present? %>
                <% params[:status].each do |s| %>
                  <%= hidden_field_tag "status[]", s %>
                <% end %>
              <% end %>

              <%= f.label :sort, "Sort:", class: "sort-label" %>
              <%= f.select :sort,
                options_for_select([
                  ["Featured", "featured"],
                  ["Newest", "newest"],
                  ["Price: Low to High", "price_asc"],
                  ["Price: High to Low", "price_desc"],
                  ["Best Selling", "best_selling"],
                  ["Name", "name"]
                ], params[:sort] || "featured"),
                {},
                class: "sort-select",
                data: { action: "change->auto-submit#submit" } %>
            <% end %>
          </div>
        </div>

        <% if @products.any? %>
          <div class="products-grid">
            <%= render partial: "products/product_card", collection: @products, as: :product %>
          </div>

          <nav class="pagination" aria-label="Product pagination">
            <%== pagy_nav(@pagy) %>
          </nav>
        <% else %>
          <div class="empty-state">
            <h3>No products found</h3>
            <p>Try adjusting your filters or search terms.</p>
            <%= link_to "Clear all filters", products_path, class: "btn btn-primary" %>
          </div>
        <% end %>
      <% end %>
    </main>
  </div>
</div>
```

### Product Card Partial

```erb
<%# app/views/products/_product_card.html.erb %>
<div class="product-card" id="<%= dom_id(product) %>">
  <% if product.image.attached? %>
    <%= image_tag product.image.variant(resize_to_fill: [300, 300]),
      class: "product-image",
      loading: "lazy",
      alt: product.name %>
  <% end %>
  <div class="product-info">
    <h3 class="product-name">
      <%= link_to product.name, product_path(product), data: { turbo_frame: "_top" } %>
    </h3>
    <p class="product-brand"><%= product.brand.name %></p>
    <p class="product-price"><%= number_to_currency(product.price) %></p>
    <% unless product.available? %>
      <span class="badge badge--warning"><%= product.status.titleize %></span>
    <% end %>
  </div>
</div>
```

### Auto-Submit Controller

```javascript
// app/javascript/controllers/auto_submit_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = {
    delay: { type: Number, default: 300 }
  }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }
}
```

---

## Example 3: Settings Page with Tabbed Sections

A user settings page with multiple tabs (Profile, Notifications, Security, Billing). Each tab loads its content into a shared frame. The active tab is tracked via URL parameter for bookmarkability.

**Patterns combined:** Tabbed navigation, Turbo Drive caching, form submission within frames

### Routes

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resource :settings, only: [:show] do
    member do
      get :profile
      get :notifications
      get :security
      get :billing
      patch :update_profile
      patch :update_notifications
      patch :update_security
    end
  end
end
```

### Controller

```ruby
# app/controllers/settings_controller.rb
class SettingsController < ApplicationController
  before_action :set_user

  TABS = %w[profile notifications security billing].freeze

  def show
    @active_tab = valid_tab(params[:tab])
    redirect_to settings_path(tab: "profile") unless params[:tab].present?
  end

  def profile
    @active_tab = "profile"
  end

  def notifications
    @active_tab = "notifications"
    @notification_preferences = @user.notification_preferences
  end

  def security
    @active_tab = "security"
  end

  def billing
    @active_tab = "billing"
    @subscription = @user.subscription
    @invoices = @user.invoices.order(created_at: :desc).limit(10)
  end

  def update_profile
    if @user.update(profile_params)
      redirect_to profile_settings_path, notice: "Profile updated."
    else
      render :profile, status: :unprocessable_entity
    end
  end

  def update_notifications
    if @user.update(notification_params)
      redirect_to notifications_settings_path, notice: "Notification preferences saved."
    else
      render :notifications, status: :unprocessable_entity
    end
  end

  def update_security
    if @user.update_with_password(security_params)
      redirect_to security_settings_path, notice: "Password changed."
    else
      render :security, status: :unprocessable_entity
    end
  end

  private

  def set_user
    @user = Current.user
  end

  def valid_tab(tab)
    TABS.include?(tab) ? tab : "profile"
  end

  def profile_params
    params.require(:user).permit(:first_name, :last_name, :email, :bio, :avatar)
  end

  def notification_params
    params.require(:user).permit(
      notification_preferences: [:email_comments, :email_mentions, :email_digest, :push_enabled]
    )
  end

  def security_params
    params.require(:user).permit(:current_password, :password, :password_confirmation)
  end
end
```

### Settings Layout View

```erb
<%# app/views/settings/show.html.erb %>
<div class="settings-page">
  <header class="settings-header">
    <h1>Settings</h1>
  </header>

  <div class="settings-layout">
    <nav class="settings-tabs" data-controller="tabs" data-tabs-active-class="tab--active">
      <%= link_to "Profile",
        profile_settings_path,
        class: "tab #{'tab--active' if @active_tab == 'profile'}",
        data: {
          turbo_frame: "settings_content",
          tabs_target: "tab",
          action: "tabs#activate"
        } %>

      <%= link_to "Notifications",
        notifications_settings_path,
        class: "tab #{'tab--active' if @active_tab == 'notifications'}",
        data: {
          turbo_frame: "settings_content",
          tabs_target: "tab",
          action: "tabs#activate"
        } %>

      <%= link_to "Security",
        security_settings_path,
        class: "tab #{'tab--active' if @active_tab == 'security'}",
        data: {
          turbo_frame: "settings_content",
          tabs_target: "tab",
          action: "tabs#activate"
        } %>

      <%= link_to "Billing",
        billing_settings_path,
        class: "tab #{'tab--active' if @active_tab == 'billing'}",
        data: {
          turbo_frame: "settings_content",
          tabs_target: "tab",
          action: "tabs#activate"
        } %>
    </nav>

    <%= turbo_frame_tag "settings_content" do %>
      <%# Default content -- redirect sends to profile %>
      <%= render "settings/profile_form" %>
    <% end %>
  </div>
</div>
```

### Tab Content Views

```erb
<%# app/views/settings/profile.html.erb %>
<%= turbo_frame_tag "settings_content" do %>
  <section class="settings-section">
    <h2>Profile</h2>
    <p class="settings-description">Update your personal information and avatar.</p>

    <%= form_with model: @user, url: update_profile_settings_path, method: :patch, class: "settings-form" do |f| %>
      <% if @user.errors.any? %>
        <div class="form-errors">
          <h3><%= pluralize(@user.errors.count, "error") %> prevented saving:</h3>
          <ul>
            <% @user.errors.full_messages.each do |msg| %>
              <li><%= msg %></li>
            <% end %>
          </ul>
        </div>
      <% end %>

      <div class="form-row">
        <div class="form-group">
          <%= f.label :first_name %>
          <%= f.text_field :first_name, class: "form-input" %>
        </div>
        <div class="form-group">
          <%= f.label :last_name %>
          <%= f.text_field :last_name, class: "form-input" %>
        </div>
      </div>

      <div class="form-group">
        <%= f.label :email %>
        <%= f.email_field :email, class: "form-input" %>
      </div>

      <div class="form-group">
        <%= f.label :bio %>
        <%= f.text_area :bio, rows: 4, class: "form-input" %>
      </div>

      <div class="form-group">
        <%= f.label :avatar %>
        <%= f.file_field :avatar, accept: "image/*", class: "form-input" %>
        <% if @user.avatar.attached? %>
          <div class="current-avatar">
            <%= image_tag @user.avatar.variant(resize_to_fill: [80, 80]), class: "avatar-preview" %>
          </div>
        <% end %>
      </div>

      <div class="form-actions">
        <%= f.submit "Save profile", class: "btn btn-primary" %>
      </div>
    <% end %>
  </section>
<% end %>
```

```erb
<%# app/views/settings/notifications.html.erb %>
<%= turbo_frame_tag "settings_content" do %>
  <section class="settings-section">
    <h2>Notifications</h2>
    <p class="settings-description">Choose how and when you want to be notified.</p>

    <%= form_with model: @user, url: update_notifications_settings_path, method: :patch, class: "settings-form" do |f| %>
      <%= f.fields_for :notification_preferences, @notification_preferences do |np| %>
        <div class="toggle-group">
          <label class="toggle-label">
            <%= np.check_box :email_comments, class: "toggle-input" %>
            <span class="toggle-text">
              <strong>Comments</strong>
              <small>Email me when someone comments on my work</small>
            </span>
          </label>
        </div>

        <div class="toggle-group">
          <label class="toggle-label">
            <%= np.check_box :email_mentions, class: "toggle-input" %>
            <span class="toggle-text">
              <strong>Mentions</strong>
              <small>Email me when I am mentioned</small>
            </span>
          </label>
        </div>

        <div class="toggle-group">
          <label class="toggle-label">
            <%= np.check_box :email_digest, class: "toggle-input" %>
            <span class="toggle-text">
              <strong>Weekly digest</strong>
              <small>Receive a weekly summary of activity</small>
            </span>
          </label>
        </div>

        <div class="toggle-group">
          <label class="toggle-label">
            <%= np.check_box :push_enabled, class: "toggle-input" %>
            <span class="toggle-text">
              <strong>Push notifications</strong>
              <small>Receive push notifications on mobile</small>
            </span>
          </label>
        </div>
      <% end %>

      <div class="form-actions">
        <%= f.submit "Save preferences", class: "btn btn-primary" %>
      </div>
    <% end %>
  </section>
<% end %>
```

```erb
<%# app/views/settings/security.html.erb %>
<%= turbo_frame_tag "settings_content" do %>
  <section class="settings-section">
    <h2>Security</h2>
    <p class="settings-description">Change your password.</p>

    <%= form_with model: @user, url: update_security_settings_path, method: :patch, class: "settings-form" do |f| %>
      <% if @user.errors.any? %>
        <div class="form-errors">
          <ul>
            <% @user.errors.full_messages.each do |msg| %>
              <li><%= msg %></li>
            <% end %>
          </ul>
        </div>
      <% end %>

      <div class="form-group">
        <%= f.label :current_password %>
        <%= f.password_field :current_password, class: "form-input", autocomplete: "current-password" %>
      </div>

      <div class="form-group">
        <%= f.label :password, "New password" %>
        <%= f.password_field :password, class: "form-input", autocomplete: "new-password" %>
      </div>

      <div class="form-group">
        <%= f.label :password_confirmation, "Confirm new password" %>
        <%= f.password_field :password_confirmation, class: "form-input", autocomplete: "new-password" %>
      </div>

      <div class="form-actions">
        <%= f.submit "Change password", class: "btn btn-primary" %>
      </div>
    <% end %>
  </section>
<% end %>
```

```erb
<%# app/views/settings/billing.html.erb %>
<%= turbo_frame_tag "settings_content" do %>
  <section class="settings-section">
    <h2>Billing</h2>
    <p class="settings-description">Manage your subscription and view invoices.</p>

    <% if @subscription %>
      <div class="billing-card">
        <h3>Current Plan</h3>
        <div class="plan-details">
          <strong><%= @subscription.plan_name %></strong>
          <span class="plan-price"><%= number_to_currency(@subscription.monthly_price) %>/month</span>
          <span class="plan-status badge badge--<%= @subscription.status %>"><%= @subscription.status.titleize %></span>
        </div>
        <% if @subscription.active? %>
          <p class="text-sm text-gray-500">
            Next billing date: <%= @subscription.next_billing_date.strftime("%B %d, %Y") %>
          </p>
        <% end %>
        <%= link_to "Change plan", "#", class: "btn btn-secondary", data: { turbo_frame: "_top" } %>
      </div>
    <% end %>

    <% if @invoices.any? %>
      <div class="invoices-section">
        <h3>Recent Invoices</h3>
        <table class="invoices-table">
          <thead>
            <tr>
              <th>Date</th>
              <th>Amount</th>
              <th>Status</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <% @invoices.each do |invoice| %>
              <tr>
                <td><%= invoice.created_at.strftime("%b %d, %Y") %></td>
                <td><%= number_to_currency(invoice.amount) %></td>
                <td><span class="badge badge--<%= invoice.status %>"><%= invoice.status %></span></td>
                <td><%= link_to "Download", invoice.pdf_url, target: "_blank", data: { turbo: false } %></td>
              </tr>
            <% end %>
          </tbody>
        </table>
      </div>
    <% end %>
  </section>
<% end %>
```

### Tabs Stimulus Controller

```javascript
// app/javascript/controllers/tabs_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["tab"]
  static classes = ["active"]

  activate(event) {
    this.tabTargets.forEach(tab => tab.classList.remove(...this.activeClasses))
    event.currentTarget.classList.add(...this.activeClasses)
  }

  connect() {
    // Set initial active tab based on current URL
    const currentPath = window.location.pathname
    this.tabTargets.forEach(tab => {
      if (tab.getAttribute("href") === currentPath) {
        this.tabTargets.forEach(t => t.classList.remove(...this.activeClasses))
        tab.classList.add(...this.activeClasses)
      }
    })
  }
}
```

### Key Implementation Notes

1. **Forms submit within the frame.** When the profile form submits, the controller redirects back to the profile tab. The redirect response includes the `settings_content` frame, so only the tab content updates -- the tab bar and page header stay in place.

2. **Validation errors render within the frame.** When `render :profile, status: :unprocessable_entity` is returned, Turbo replaces the frame content with the form including error messages. The 422 status tells Turbo to render the response rather than follow a redirect.

3. **External links break out of the frame.** The "Change plan" link and invoice download links use `data: { turbo_frame: "_top" }` or `data: { turbo: false }` to navigate outside the frame context.

4. **Each tab has its own URL.** Users can bookmark `settings/security` directly. When visiting that URL without JavaScript, the full page renders with the security tab content.
