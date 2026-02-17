---
title: "Practical Form Patterns"
categories:
  - forms
  - examples
  - patterns
tags:
  - inline-edit
  - modal
  - wizard
  - turbo-frames
  - turbo-streams
  - stimulus
description: >-
  Practical form patterns combining multiple techniques: inline editable table
  rows, modal new-record forms with list append, and multi-step wizard forms
  with step frames and state preservation.
---

# Practical Form Patterns

Three complete examples that combine Turbo Frames, Turbo Streams, Stimulus, and proper form lifecycle handling into production-ready patterns.

## Table of Contents

- [Example 1: Inline Editable Table Row](#example-1-inline-editable-table-row)
- [Example 2: Modal New Record Form](#example-2-modal-new-record-form)
- [Example 3: Multi-Step Wizard Form](#example-3-multi-step-wizard-form)

---

## Example 1: Inline Editable Table Row

**Scenario**: A task list displayed as a table. Each row can be clicked to edit inline. The edit form replaces the row, and saving or cancelling restores the display row. Blur-to-save for a fluid experience.

### Routes

```ruby
# config/routes.rb
resources :tasks, only: %i[ index show edit update ]
```

### Model

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  validates :title, presence: true
  validates :due_date, presence: true

  scope :chronologically, -> { order(due_date: :asc) }
end
```

### Controller

```ruby
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  before_action :set_task, only: %i[ show edit update ]

  def index
    @tasks = Task.chronologically
  end

  def show
    # Renders the display partial in the frame
  end

  def edit
    # Renders the edit form in the frame
  end

  def update
    if @task.update(task_params)
      redirect_to @task, status: :see_other
    else
      render :edit, status: :unprocessable_entity
    end
  end

  private
    def set_task
      @task = Task.find(params[:id])
    end

    def task_params
      params.require(:task).permit(:title, :due_date, :priority)
    end
end
```

### Views

```erb
<%# app/views/tasks/index.html.erb %>
<h1 class="text-2xl font-bold mb-6">Tasks</h1>

<table class="w-full">
  <thead>
    <tr class="text-left text-sm text-gray-500 border-b">
      <th class="pb-2">Task</th>
      <th class="pb-2">Due Date</th>
      <th class="pb-2">Priority</th>
      <th class="pb-2"></th>
    </tr>
  </thead>
  <tbody>
    <% @tasks.each do |task| %>
      <%= render task %>
    <% end %>
  </tbody>
</table>
```

```erb
<%# app/views/tasks/_task.html.erb %>
<%= turbo_frame_tag dom_id(task), tag: "tr", class: "border-b group hover:bg-gray-50" do %>
  <td class="py-3"><%= task.title %></td>
  <td class="py-3 text-sm text-gray-600"><%= task.due_date.strftime("%b %d, %Y") %></td>
  <td class="py-3">
    <span class="px-2 py-1 text-xs rounded-full
                 <%= task.priority == 'high' ? 'bg-red-100 text-red-800' : 'bg-gray-100 text-gray-800' %>">
      <%= task.priority %>
    </span>
  </td>
  <td class="py-3 text-right">
    <%= link_to "Edit", edit_task_path(task),
          class: "text-sm text-blue-600 opacity-0 group-hover:opacity-100 transition" %>
  </td>
<% end %>
```

```erb
<%# app/views/tasks/show.html.erb %>
<%= render @task %>
```

```erb
<%# app/views/tasks/edit.html.erb %>
<%= turbo_frame_tag dom_id(@task), tag: "tr" do %>
  <%= form_with model: @task,
        class: "contents",
        data: { controller: "inline-edit" } do |f| %>
    <td class="py-2">
      <%= f.text_field :title,
            class: "w-full rounded border-gray-300 text-sm
                    #{'border-red-500' if @task.errors[:title].any?}",
            autofocus: true,
            data: { action: "blur->inline-edit#save keydown.escape->inline-edit#cancel" } %>
      <% if @task.errors[:title].any? %>
        <p class="text-xs text-red-600 mt-1"><%= @task.errors[:title].first %></p>
      <% end %>
    </td>
    <td class="py-2">
      <%= f.date_field :due_date,
            class: "rounded border-gray-300 text-sm",
            data: { action: "change->inline-edit#save" } %>
    </td>
    <td class="py-2">
      <%= f.select :priority, %w[low medium high],
            {},
            { class: "rounded border-gray-300 text-sm",
              data: { action: "change->inline-edit#save" } } %>
    </td>
    <td class="py-2 text-right">
      <%= link_to "Cancel", task_path(@task), class: "text-sm text-gray-500" %>
    </td>
  <% end %>
<% end %>
```

### Stimulus Controller

```javascript
// app/javascript/controllers/inline_edit_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  save() {
    this.element.requestSubmit()
  }

  cancel(event) {
    event.preventDefault()
    // Click the cancel link to navigate back to the display state
    this.element.querySelector("a[href]")?.click()
  }
}
```

---

## Example 2: Modal New Record Form

**Scenario**: A project page shows a list of tasks. Clicking "New Task" opens a modal dialog with a form. Validation errors re-render inside the modal. On success, the modal closes and the new task is appended to the list without a full page reload.

### Routes

```ruby
# config/routes.rb
resources :projects, only: :show do
  resources :tasks, only: %i[ new create ]
end
```

### Controller

```ruby
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  before_action :set_project

  def new
    @task = @project.tasks.build
  end

  def create
    @task = @project.tasks.build(task_params)
    if @task.save
      respond_to do |format|
        format.turbo_stream
        format.html { redirect_to @project, status: :see_other }
      end
    else
      render :new, status: :unprocessable_entity
    end
  end

  private
    def set_project
      @project = Project.find(params[:project_id])
    end

    def task_params
      params.require(:task).permit(:title, :description, :priority)
    end
end
```

### Views

```erb
<%# app/views/projects/show.html.erb %>
<div class="max-w-4xl mx-auto py-8">
  <div class="flex items-center justify-between mb-6">
    <h1 class="text-2xl font-bold"><%= @project.name %></h1>
    <%= link_to "New Task",
          new_project_task_path(@project),
          data: { turbo_frame: "modal" },
          class: "px-4 py-2 bg-blue-600 text-white rounded-md text-sm" %>
  </div>

  <div id="tasks_list" class="space-y-2">
    <%= render @project.tasks.chronologically %>
  </div>
</div>

<%# Modal dialog (reusable across pages) %>
<dialog data-controller="modal"
        data-modal-target="dialog"
        data-action="close->modal#cleanup click->modal#clickOutside"
        class="backdrop:bg-black/50 rounded-lg shadow-xl p-0 max-w-lg w-full">
  <%= turbo_frame_tag "modal",
        data: { action: "turbo:frame-load->modal#open" } do %>
  <% end %>
</dialog>
```

```erb
<%# app/views/tasks/new.html.erb %>
<%= turbo_frame_tag "modal" do %>
  <div class="p-6">
    <div class="flex items-center justify-between mb-4">
      <h2 class="text-lg font-semibold">New Task</h2>
      <button type="button"
              data-action="modal#close"
              class="text-gray-400 hover:text-gray-600 text-xl">&times;</button>
    </div>

    <% if @task.errors.any? %>
      <div class="rounded-md bg-red-50 p-3 mb-4">
        <ul class="list-disc list-inside text-sm text-red-700">
          <% @task.errors.full_messages.each do |message| %>
            <li><%= message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <%= form_with model: [@project, @task] do |f| %>
      <div class="space-y-4">
        <div>
          <%= f.label :title, class: "block text-sm font-medium text-gray-700" %>
          <%= f.text_field :title,
                autofocus: true,
                class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500
                        #{'border-red-500' if @task.errors[:title].any?}" %>
        </div>

        <div>
          <%= f.label :description, class: "block text-sm font-medium text-gray-700" %>
          <%= f.text_area :description,
                rows: 3,
                class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring-blue-500" %>
        </div>

        <div>
          <%= f.label :priority, class: "block text-sm font-medium text-gray-700" %>
          <%= f.select :priority, %w[low medium high],
                {},
                { class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm" } %>
        </div>
      </div>

      <div class="mt-6 flex justify-end gap-3">
        <button type="button"
                data-action="modal#close"
                class="px-4 py-2 text-sm text-gray-700 hover:text-gray-900">
          Cancel
        </button>
        <%= f.submit "Create Task",
              class: "px-4 py-2 bg-blue-600 text-white rounded-md text-sm hover:bg-blue-700",
              data: { turbo_submits_with: "Creating..." } %>
      </div>
    <% end %>
  </div>
<% end %>
```

```erb
<%# app/views/tasks/create.turbo_stream.erb %>
<%# Append the new task to the list %>
<%= turbo_stream.append "tasks_list", partial: "tasks/task", locals: { task: @task } %>

<%# Clear the modal frame to trigger close %>
<%= turbo_stream.update "modal", "" %>
```

```erb
<%# app/views/tasks/_task.html.erb %>
<div id="<%= dom_id(task) %>" class="flex items-center justify-between p-3 bg-white rounded-md border border-gray-200">
  <div>
    <p class="font-medium"><%= task.title %></p>
    <p class="text-sm text-gray-500"><%= task.description&.truncate(80) %></p>
  </div>
  <span class="px-2 py-1 text-xs rounded-full
               <%= task.priority == 'high' ? 'bg-red-100 text-red-800' : 'bg-gray-100 text-gray-800' %>">
    <%= task.priority %>
  </span>
</div>
```

### Modal Stimulus Controller

```javascript
// app/javascript/controllers/modal_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["dialog"]

  open() {
    this.dialogTarget.showModal()
  }

  close() {
    this.dialogTarget.close()
  }

  cleanup() {
    // Clear frame content so next modal starts fresh
    const frame = this.dialogTarget.querySelector("turbo-frame")
    if (frame) {
      frame.innerHTML = ""
      frame.removeAttribute("src")
    }
  }

  clickOutside(event) {
    if (event.target === this.dialogTarget) {
      this.close()
    }
  }
}
```

---

## Example 3: Multi-Step Wizard Form

**Scenario**: A multi-step registration wizard. Each step is a Turbo Frame that loads the next step's form. The wizard preserves state across steps and allows navigating back to previous steps.

### Routes

```ruby
# config/routes.rb
resource :registration, only: %i[ new create ] do
  member do
    get :step_1
    get :step_2
    get :step_3
    post :complete
  end
end
```

### Controller

```ruby
# app/controllers/registrations_controller.rb
class RegistrationsController < ApplicationController
  before_action :load_registration_from_session

  def new
    redirect_to step_1_registration_path
  end

  def step_1
    # Personal information
  end

  def step_2
    # Account details
  end

  def step_3
    # Confirmation / review
  end

  def create
    case params[:step]
    when "1"
      save_to_session(:step_1, step_1_params)
      if valid_step_1?
        redirect_to step_2_registration_path, status: :see_other
      else
        render :step_1, status: :unprocessable_entity
      end
    when "2"
      save_to_session(:step_2, step_2_params)
      if valid_step_2?
        redirect_to step_3_registration_path, status: :see_other
      else
        render :step_2, status: :unprocessable_entity
      end
    end
  end

  def complete
    @user = User.new(registration_params_from_session)
    if @user.save
      session.delete(:registration)
      redirect_to dashboard_path, status: :see_other, notice: "Welcome!"
    else
      @errors = @user.errors
      render :step_3, status: :unprocessable_entity
    end
  end

  private
    def load_registration_from_session
      @registration = session[:registration] || {}
    end

    def save_to_session(step, params)
      session[:registration] ||= {}
      session[:registration][step] = params.to_h
    end

    def valid_step_1?
      params[:registration][:first_name].present? &&
        params[:registration][:last_name].present? &&
        params[:registration][:email].present?
    end

    def valid_step_2?
      params[:registration][:username].present? &&
        params[:registration][:password].present?
    end

    def registration_params_from_session
      (session.dig(:registration, :step_1) || {})
        .merge(session.dig(:registration, :step_2) || {})
    end

    def step_1_params
      params.require(:registration).permit(:first_name, :last_name, :email)
    end

    def step_2_params
      params.require(:registration).permit(:username, :password)
    end
end
```

### Views

```erb
<%# app/views/registrations/_progress.html.erb %>
<nav class="flex items-center justify-center gap-8 mb-8">
  <% [
    { step: 1, label: "Personal Info", path: step_1_registration_path },
    { step: 2, label: "Account", path: step_2_registration_path },
    { step: 3, label: "Confirm", path: step_3_registration_path }
  ].each do |item| %>
    <div class="flex items-center gap-2
                <%= item[:step] == current_step ? 'text-blue-600 font-semibold' : 'text-gray-400' %>
                <%= item[:step] < current_step ? 'text-green-600' : '' %>">
      <span class="w-8 h-8 rounded-full flex items-center justify-center text-sm border-2
                   <%= item[:step] == current_step ? 'border-blue-600 bg-blue-50' : '' %>
                   <%= item[:step] < current_step ? 'border-green-600 bg-green-50' : 'border-gray-300' %>">
        <%= item[:step] < current_step ? '&#10003;'.html_safe : item[:step] %>
      </span>
      <span class="text-sm"><%= item[:label] %></span>
    </div>
  <% end %>
</nav>
```

```erb
<%# app/views/registrations/step_1.html.erb %>
<div class="max-w-md mx-auto py-12">
  <%= render "progress", current_step: 1 %>

  <%= turbo_frame_tag "wizard_step" do %>
    <h2 class="text-xl font-semibold mb-6">Personal Information</h2>

    <%= form_with url: registration_path, method: :post do |f| %>
      <%= hidden_field_tag :step, 1 %>

      <div class="space-y-4">
        <div>
          <%= f.label :first_name, "First Name", class: "block text-sm font-medium text-gray-700",
                for: "registration_first_name" %>
          <%= text_field_tag "registration[first_name]",
                @registration.dig(:step_1, "first_name"),
                id: "registration_first_name",
                autofocus: true,
                class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm" %>
        </div>

        <div>
          <%= f.label :last_name, "Last Name", class: "block text-sm font-medium text-gray-700",
                for: "registration_last_name" %>
          <%= text_field_tag "registration[last_name]",
                @registration.dig(:step_1, "last_name"),
                id: "registration_last_name",
                class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm" %>
        </div>

        <div>
          <%= f.label :email, "Email", class: "block text-sm font-medium text-gray-700",
                for: "registration_email" %>
          <%= email_field_tag "registration[email]",
                @registration.dig(:step_1, "email"),
                id: "registration_email",
                class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm" %>
        </div>
      </div>

      <div class="mt-8 flex justify-end">
        <%= f.submit "Next: Account Details",
              class: "px-6 py-2 bg-blue-600 text-white rounded-md",
              data: { turbo_submits_with: "Saving..." } %>
      </div>
    <% end %>
  <% end %>
</div>
```

```erb
<%# app/views/registrations/step_2.html.erb %>
<div class="max-w-md mx-auto py-12">
  <%= render "progress", current_step: 2 %>

  <%= turbo_frame_tag "wizard_step" do %>
    <h2 class="text-xl font-semibold mb-6">Account Details</h2>

    <%= form_with url: registration_path, method: :post do |f| %>
      <%= hidden_field_tag :step, 2 %>

      <div class="space-y-4">
        <div>
          <%= f.label :username, "Username", class: "block text-sm font-medium text-gray-700",
                for: "registration_username" %>
          <%= text_field_tag "registration[username]",
                @registration.dig(:step_2, "username"),
                id: "registration_username",
                autofocus: true,
                class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm" %>
        </div>

        <div>
          <%= f.label :password, "Password", class: "block text-sm font-medium text-gray-700",
                for: "registration_password" %>
          <%= password_field_tag "registration[password]",
                nil,
                id: "registration_password",
                class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm" %>
        </div>
      </div>

      <div class="mt-8 flex justify-between">
        <%= link_to "Back",
              step_1_registration_path,
              class: "px-6 py-2 text-gray-600 hover:text-gray-900" %>
        <%= f.submit "Next: Review",
              class: "px-6 py-2 bg-blue-600 text-white rounded-md",
              data: { turbo_submits_with: "Saving..." } %>
      </div>
    <% end %>
  <% end %>
</div>
```

```erb
<%# app/views/registrations/step_3.html.erb %>
<div class="max-w-md mx-auto py-12">
  <%= render "progress", current_step: 3 %>

  <%= turbo_frame_tag "wizard_step" do %>
    <h2 class="text-xl font-semibold mb-6">Review & Confirm</h2>

    <% if @errors&.any? %>
      <div class="rounded-md bg-red-50 p-4 mb-6">
        <ul class="list-disc list-inside text-sm text-red-700">
          <% @errors.full_messages.each do |message| %>
            <li><%= message %></li>
          <% end %>
        </ul>
      </div>
    <% end %>

    <div class="space-y-4 bg-gray-50 rounded-lg p-4">
      <div class="flex justify-between">
        <span class="text-sm text-gray-500">Name</span>
        <span class="text-sm font-medium">
          <%= @registration.dig(:step_1, "first_name") %> <%= @registration.dig(:step_1, "last_name") %>
        </span>
      </div>
      <div class="flex justify-between">
        <span class="text-sm text-gray-500">Email</span>
        <span class="text-sm font-medium"><%= @registration.dig(:step_1, "email") %></span>
      </div>
      <div class="flex justify-between">
        <span class="text-sm text-gray-500">Username</span>
        <span class="text-sm font-medium"><%= @registration.dig(:step_2, "username") %></span>
      </div>
    </div>

    <div class="mt-8 flex justify-between">
      <%= link_to "Back",
            step_2_registration_path,
            class: "px-6 py-2 text-gray-600 hover:text-gray-900" %>
      <%= button_to "Create Account",
            complete_registration_path,
            method: :post,
            class: "px-6 py-2 bg-green-600 text-white rounded-md",
            data: { turbo_submits_with: "Creating Account..." } %>
    </div>
  <% end %>
</div>
```

### Key Points

- **Session storage**: Wizard state is stored in the server session, not in hidden fields. This prevents tampering and allows the user to navigate back to previous steps without losing data.
- **422/303 lifecycle**: Each step validates and either re-renders (422) or redirects to the next step (303).
- **Turbo Frame for steps**: The `wizard_step` frame wraps each step, so only the step content changes when navigating. The progress bar is outside the frame and updates on each full page load.
- **Back navigation**: "Back" links are regular links that navigate within the frame (or as full page loads if outside the frame). Session data persists so the user sees their previously entered values.
- **Final validation**: The `complete` action builds the actual User record from session data and performs full model validation. If it fails, the user sees errors on the review step.
