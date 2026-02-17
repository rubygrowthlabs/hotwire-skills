---
title: "Rails Native Backend"
categories:
  - hotwire-native
  - rails
  - backend
tags:
  - turbo-native-app
  - conditional-rendering
  - path-configuration
  - turbo-streams
  - versioning
description: >-
  Rails backend patterns for Hotwire Native: turbo_native_app? user agent detection,
  conditional rendering for native vs web layouts, path configuration endpoint, form
  submission patterns with recede/refresh/resume, native-specific Turbo Stream responses,
  and minimum app version enforcement.
---

# Rails Native Backend

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [User Agent Detection With turbo_native_app?](#user-agent-detection-with-turbo_native_app)
  - [Conditional Rendering for Native vs Web](#conditional-rendering-for-native-vs-web)
  - [Path Configuration Endpoint](#path-configuration-endpoint)
  - [Form Patterns: Recede, Refresh, Resume](#form-patterns-recede-refresh-resume)
  - [Native-Specific Turbo Stream Responses](#native-specific-turbo-stream-responses)
  - [App Version Enforcement](#app-version-enforcement)
- [Pattern Card](#pattern-card)

## Overview

The Rails backend is the brain of a Hotwire Native app. Since the native app renders server-side HTML, the Rails app controls not only business logic and data but also navigation behavior, layout, and feature availability for native clients.

The key server-side patterns are:
1. **Detect native clients** using the `turbo_native_app?` helper based on user agent.
2. **Render conditionally** with native-specific layouts that hide web-only chrome (navigation bar, footer).
3. **Serve path configuration** as a JSON endpoint for server-driven routing.
4. **Handle form submissions** with redirect patterns that work with native navigation (recede, refresh, resume).
5. **Enforce minimum app versions** to ensure compatibility with server changes.

The guiding principle: the Rails app should serve a single HTML page that works in both a browser and a native shell. Use `turbo_native_app?` to conditionally enhance, never to gate functionality.

## Implementation

### User Agent Detection With turbo_native_app?

Rails provides a built-in helper that checks whether the request comes from a Hotwire Native app. It works by matching "Turbo Native" in the user agent string.

```ruby
# Available in controllers and views automatically
turbo_native_app?  # => true or false
```

This helper is available because Hotwire Native SDKs append "Turbo Native" to the web view's user agent string (configured during native app setup).

Use it in controllers:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  helper_method :turbo_native_app?

  private

  # Layout selection based on client type
  def set_native_layout
    if turbo_native_app?
      "native"
    else
      "application"
    end
  end
end
```

Use it in views:

```erb
<%# Show web navigation only in browser, not in native shell %>
<% unless turbo_native_app? %>
  <nav class="main-navigation">
    <%= link_to "Home", root_path %>
    <%= link_to "Profile", profile_path %>
  </nav>
<% end %>
```

### Conditional Rendering for Native vs Web

Create a dedicated layout for native clients that strips out web-only elements (navigation bar, footer, cookie banners) that the native shell replaces with native equivalents.

**Native layout:**

```erb
<%# app/views/layouts/native.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <title><%= content_for(:title) || "MyApp" %></title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <%= csrf_meta_tags %>
  <%= csp_meta_tag %>
  <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
  <%= javascript_include_tag "application", "data-turbo-track": "reload", defer: true %>
</head>
<body class="native">
  <%# No web navigation bar -- native shell provides this %>
  <%# No footer -- native tab bar replaces this %>
  <%# No cookie banner -- not applicable in native app %>

  <main>
    <%= yield %>
  </main>
</body>
</html>
```

**Switching layouts per controller:**

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  layout :resolve_layout

  private

  def resolve_layout
    turbo_native_app? ? "native" : "application"
  end
end
```

**Conditional partials for specific elements:**

```erb
<%# app/views/posts/show.html.erb %>
<article>
  <h1><%= @post.title %></h1>
  <%= simple_format(@post.body) %>

  <div class="actions">
    <%# Web: show all action buttons inline %>
    <%# Native: hide web buttons, bridge component provides native bar buttons %>
    <div class="<%= 'hidden' if turbo_native_app? %>">
      <%= link_to "Edit", edit_post_path(@post), class: "btn" %>
      <%= link_to "Share", "#", class: "btn", data: { action: "click->share#open" } %>
    </div>

    <%# Bridge component data for native menu (only consumed by native app) %>
    <div data-controller="bridge--menu"
         data-bridge--menu-items-value='<%= [
           { title: "Edit", icon: "pencil", event: "edit" },
           { title: "Share", icon: "square.and.arrow.up", event: "share" },
           { title: "Delete", icon: "trash", event: "delete", destructive: true }
         ].to_json %>'>
    </div>
  </div>
</article>
```

### Path Configuration Endpoint

Serve the path configuration JSON from a dedicated controller. This is the central routing table that native apps fetch on launch.

```ruby
# app/controllers/api/v1/turbo/path_configurations_controller.rb
module Api
  module V1
    module Turbo
      class PathConfigurationsController < ApplicationController
        # No authentication required -- native app needs this before login
        skip_before_action :authenticate_user!, if: -> { defined?(authenticate_user!) }
        skip_before_action :verify_authenticity_token

        def show
          # Cache for 5 minutes to reduce server load
          expires_in 5.minutes, public: true

          render json: {
            settings: settings,
            rules: rules
          }
        end

        private

        def settings
          {
            tabs: [
              { title: "Home", path: "/", icon: "house" },
              { title: "Search", path: "/search", icon: "magnifyingglass" },
              { title: "Notifications", path: "/notifications", icon: "bell" },
              { title: "Profile", path: "/profile", icon: "person" }
            ],
            minimum_app_version: "1.2.0",
            maintenance_mode: false
          }
        end

        def rules
          [
            # Default: push web view with pull-to-refresh
            {
              patterns: [".*"],
              properties: {
                context: "default",
                presentation: "default",
                pull_to_refresh_enabled: true
              }
            },
            # Forms open as modals without pull-to-refresh
            {
              patterns: ["/new$", "/edit$", "/.*/edit$"],
              properties: {
                context: "modal",
                presentation: "default",
                pull_to_refresh_enabled: false
              }
            },
            # Auth screens as modal with replace root after success
            {
              patterns: ["/login", "/sign_in", "/users/sign_in", "/sign_up"],
              properties: {
                context: "modal",
                presentation: "replace_root",
                pull_to_refresh_enabled: false
              }
            },
            # Native camera screen
            {
              patterns: ["/camera"],
              properties: {
                view_controller: "native_camera",
                presentation: "modal"
              }
            },
            # Logout clears all navigation stacks
            {
              patterns: ["/logout", "/sign_out", "/users/sign_out"],
              properties: {
                presentation: "clear_all"
              }
            }
          ]
        end
      end
    end
  end
end
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      namespace :turbo do
        resource :path_configuration, only: :show
      end
    end
  end
end
```

### Form Patterns: Recede, Refresh, Resume

When a form submission succeeds in a Hotwire Native app, the native shell needs to know what to do next. There are three patterns:

| Pattern | Behavior | When to Use |
|---------|----------|-------------|
| **Recede** | Pop the modal/screen and go back | Creating a new record from a modal |
| **Refresh** | Reload the current page | Updating an existing record inline |
| **Resume** | Stay on the current page | Multi-step forms, wizard flows |

These patterns are controlled by the HTTP redirect after form submission:

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = Current.user.posts.build(post_params)

    if @post.save
      # Recede: redirect back to the list (native will pop the modal)
      redirect_to posts_path, notice: "Post created!"
    else
      render :new, status: :unprocessable_entity
    end
  end

  def update
    @post = Current.user.posts.find(params[:id])

    if @post.update(post_params)
      # Refresh: redirect to the same resource (native reloads in place)
      redirect_to @post, notice: "Post updated!"
    else
      render :edit, status: :unprocessable_entity
    end
  end
end
```

The key is how Turbo handles redirects:
- **Redirect to a different URL** (e.g., `posts_path` after create): Turbo performs an "advance" visit. If the current screen was a modal (per path config), it is dismissed, and the previous screen navigates to the redirect URL. This is the "recede" pattern.
- **Redirect to the same URL** (e.g., `@post` after update): Turbo performs a "replace" visit, reloading the page in place. This is the "refresh" pattern.

For explicit control, use `data-turbo-action` on the form:

```erb
<%# Force the form to perform a specific navigation action after submission %>
<%= form_with(model: @post, data: { turbo_action: "advance" }) do |f| %>
  <%# ... fields ... %>
<% end %>
```

### Native-Specific Turbo Stream Responses

Sometimes you want different Turbo Stream behavior for native vs web clients. For example, after creating a record, the web app might append it to a list, but the native app should dismiss the modal.

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = Current.user.posts.build(post_params)

    if @post.save
      respond_to do |format|
        format.html { redirect_to posts_path, notice: "Post created!" }
        format.turbo_stream do
          if turbo_native_app?
            # Native: redirect to dismiss modal and refresh the list
            redirect_to posts_path, notice: "Post created!"
          else
            # Web: append the new post to the list inline
            render turbo_stream: turbo_stream.prepend(
              "posts",
              partial: "posts/post",
              locals: { post: @post }
            )
          end
        end
      end
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

For flash messages in native apps, use a bridge component instead of web-based flash UI:

```erb
<%# app/views/layouts/native.html.erb %>
<% if flash[:notice] %>
  <div data-controller="bridge--alert"
       data-bridge--alert-title-value="Success"
       data-bridge--alert-message-value="<%= flash[:notice] %>">
  </div>
<% end %>
```

### App Version Enforcement

Enforce a minimum app version to prevent old native clients from accessing the server after breaking changes. The native app sends its version in the user agent; the server checks it.

```ruby
# app/controllers/concerns/native_version_check.rb
module NativeVersionCheck
  extend ActiveSupport::Concern

  MINIMUM_IOS_VERSION = "1.2.0"
  MINIMUM_ANDROID_VERSION = "1.2.0"

  included do
    before_action :check_native_app_version, if: :turbo_native_app?
  end

  private

  def check_native_app_version
    version = extract_app_version
    return unless version

    minimum = if request.user_agent.include?("iOS")
                MINIMUM_IOS_VERSION
              elsif request.user_agent.include?("Android")
                MINIMUM_ANDROID_VERSION
              end

    return unless minimum

    if Gem::Version.new(version) < Gem::Version.new(minimum)
      render "shared/upgrade_required", layout: "native", status: :upgrade_required
    end
  end

  def extract_app_version
    # Expects user agent format: "MyApp/1.2.0 Turbo Native iOS ..."
    match = request.user_agent.match(/MyApp\/([\d.]+)/)
    match&.[](1)
  end
end
```

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include NativeVersionCheck
end
```

```erb
<%# app/views/shared/upgrade_required.html.erb %>
<div class="upgrade-required">
  <h1>Update Required</h1>
  <p>Please update MyApp to the latest version to continue.</p>

  <% if request.user_agent.include?("iOS") %>
    <%= link_to "Update on App Store",
                "https://apps.apple.com/app/myapp/id123456789",
                class: "btn btn-primary" %>
  <% else %>
    <%= link_to "Update on Google Play",
                "https://play.google.com/store/apps/details?id=com.example.myapp",
                class: "btn btn-primary" %>
  <% end %>
</div>
```

You can also include the minimum version in path configuration settings so the native app can check proactively:

```json
{
  "settings": {
    "minimum_app_version": "1.2.0"
  }
}
```

## Pattern Card

### GOOD: turbo_native_app? Conditional Rendering With Shared Templates

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  layout :resolve_layout

  private

  def resolve_layout
    turbo_native_app? ? "native" : "application"
  end
end
```

```erb
<%# app/views/posts/show.html.erb -- SAME template for web and native %>
<article>
  <h1><%= @post.title %></h1>
  <%= simple_format(@post.body) %>

  <%# Web buttons (hidden in native, replaced by bridge component) %>
  <div class="<%= 'hidden' if turbo_native_app? %>">
    <%= link_to "Edit", edit_post_path(@post) %>
  </div>

  <%# Bridge component data (ignored in web, consumed in native) %>
  <div data-controller="bridge--form-submit">
  </div>
</article>
```

```ruby
# Same controller handles both web and native
def create
  @post = Current.user.posts.build(post_params)
  if @post.save
    redirect_to posts_path, notice: "Post created!"
  else
    render :new, status: :unprocessable_entity
  end
end
```

This approach uses a single set of templates, controllers, and routes for both web and native. The native layout strips web-only chrome. Bridge component data attributes are ignored by web browsers and consumed by the native app. Form submissions use standard redirects that Turbo handles correctly in both contexts.

### BAD: Separate API for Native App

```ruby
# BAD: Duplicating the entire backend for native clients
# app/controllers/api/v1/posts_controller.rb
module Api
  module V1
    class PostsController < ApiController
      # Separate authentication (JWT instead of cookies)
      before_action :authenticate_with_token!

      def index
        @posts = Post.all
        # Separate serialization (JSON instead of HTML)
        render json: PostSerializer.new(@posts)
      end

      def create
        @post = current_user.posts.build(post_params)
        if @post.save
          # Separate response format
          render json: PostSerializer.new(@post), status: :created
        else
          # Separate error format
          render json: { errors: @post.errors }, status: :unprocessable_entity
        end
      end
    end
  end
end
```

This approach duplicates the entire backend: separate controllers, separate authentication, separate serializers, separate error handling, and separate routes. Every feature must be implemented twice. Bugs must be fixed in two places. The web and native apps inevitably drift out of sync. This is the exact problem Hotwire Native was built to eliminate.
