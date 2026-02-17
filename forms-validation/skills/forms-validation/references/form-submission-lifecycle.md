---
title: "The Turbo Form Submission Lifecycle"
categories:
  - turbo
  - forms
  - lifecycle
tags:
  - turbo-drive
  - form-submission
  - 422-status
  - 303-redirect
  - turbo-submit-start
  - turbo-submit-end
  - data-turbo-submits-with
description: >-
  Complete lifecycle of a Turbo form submission from submit to response rendering.
  Covers turbo:submit-start and turbo:submit-end events, 422 vs 303 status codes,
  error re-rendering, disabling submit buttons during submission, and the
  data-turbo-submits-with attribute for button state management.
---

# The Turbo Form Submission Lifecycle

## Table of Contents

- [Overview](#overview)
- [The Complete Lifecycle](#the-complete-lifecycle)
- [Implementation](#implementation)
  - [Status Codes: 422 vs 303](#status-codes-422-vs-303)
  - [Error Re-Rendering with 422](#error-re-rendering-with-422)
  - [Success Redirect with 303](#success-redirect-with-303)
  - [Turbo Submission Events](#turbo-submission-events)
  - [Disabling Submit Buttons During Submission](#disabling-submit-buttons-during-submission)
  - [data-turbo-submits-with for Button Text](#data-turbo-submits-with-for-button-text)
  - [Handling Non-GET Form Submissions in Frames](#handling-non-get-form-submissions-in-frames)
- [Common Pitfalls](#common-pitfalls)
- [Pattern Card](#pattern-card)

## Overview

Every form submission in a Turbo-enabled Rails application follows a specific lifecycle. Understanding this lifecycle is essential because Turbo's behavior differs from traditional Rails form handling in one critical way: **the HTTP status code determines what Turbo does with the response**.

- **422 Unprocessable Entity**: Turbo renders the response body in place (re-renders the form with errors).
- **303 See Other**: Turbo follows the redirect and navigates to the new URL.
- **200 OK**: Turbo treats this as a successful response and renders it as a new page visit -- which is almost never what you want for a form submission.

Getting the status codes wrong is the single most common source of form-related bugs in Turbo applications.

## The Complete Lifecycle

```
User clicks submit
       |
       v
turbo:submit-start
  - form gets [aria-busy="true"]
  - submit button state changes (if data-turbo-submits-with)
       |
       v
Turbo sends fetch() request
  - Method: POST/PATCH/DELETE
  - Headers: Accept: text/vnd.turbo-stream.html, text/html
  - Body: FormData
       |
       v
Server processes request
       |
       +---> Validation fails
       |         |
       |         v
       |     render :action, status: :unprocessable_entity (422)
       |         |
       |         v
       |     turbo:submit-end (success: false)
       |         |
       |         v
       |     Turbo renders response body in place
       |     (form re-renders with error messages)
       |
       +---> Validation succeeds
                 |
                 v
             redirect_to path, status: :see_other (303)
                 |
                 v
             turbo:submit-end (success: true)
                 |
                 v
             Turbo follows redirect (GET request)
                 |
                 v
             turbo:load (new page rendered)
```

## Implementation

### Status Codes: 422 vs 303

This is the most important rule in Turbo form handling. Every controller action that processes a form submission must return the correct status code.

```ruby
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  def create
    @task = @project.tasks.build(task_params)
    if @task.save
      # 303 See Other -- Turbo follows this redirect
      redirect_to project_tasks_path(@project), status: :see_other
    else
      # 422 Unprocessable Entity -- Turbo re-renders the form with errors
      render :new, status: :unprocessable_entity
    end
  end

  def update
    if @task.update(task_params)
      redirect_to @task, status: :see_other
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @task.destroy!
    # 303 for DELETE actions too
    redirect_to project_tasks_path(@project), status: :see_other
  end

  private
    def task_params
      params.require(:task).permit(:title, :description)
    end
end
```

### Error Re-Rendering with 422

When the server returns 422, Turbo takes the response body and renders it in place. For a form inside a Turbo Frame, this means the frame content is replaced. For a full-page form, the entire body is replaced.

```erb
<%# app/views/tasks/new.html.erb %>
<%= form_with model: @task do |f| %>
  <% if @task.errors.any? %>
    <div class="rounded-md bg-red-50 p-4 mb-6">
      <h3 class="text-sm font-medium text-red-800">
        <%= pluralize(@task.errors.count, "error") %> prevented this task from being saved:
      </h3>
      <ul class="mt-2 list-disc list-inside text-sm text-red-700">
        <% @task.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="space-y-4">
    <div>
      <%= f.label :title, class: "block text-sm font-medium text-gray-700" %>
      <%= f.text_field :title,
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm
                    #{'border-red-500' if @task.errors[:title].any?}" %>
    </div>

    <div>
      <%= f.label :description, class: "block text-sm font-medium text-gray-700" %>
      <%= f.text_area :description,
            rows: 4,
            class: "mt-1 block w-full rounded-md border-gray-300 shadow-sm" %>
    </div>
  </div>

  <div class="mt-6">
    <%= f.submit "Create Task",
          class: "px-4 py-2 bg-blue-600 text-white rounded-md",
          data: { turbo_submits_with: "Creating..." } %>
  </div>
<% end %>
```

The form re-renders with the submitted values preserved and error messages displayed. The user can correct their input and resubmit.

### Success Redirect with 303

When the server returns 303, Turbo follows the redirect by making a GET request to the specified URL. This behaves like a normal page navigation.

```ruby
# The redirect chain:
# 1. POST /tasks (form submission)
# 2. 303 See Other -> Location: /projects/1/tasks
# 3. GET /projects/1/tasks (Turbo navigates here)
redirect_to project_tasks_path(@project), status: :see_other
```

If the form is inside a Turbo Frame, the redirect response must contain a matching frame, or Turbo will display an error. To break out of the frame and navigate the full page, use `_top`:

```ruby
# In the controller, or use data-turbo-frame="_top" on the form
redirect_to project_tasks_path(@project), status: :see_other
```

```erb
<%# If the form is in a frame but the redirect should navigate the full page %>
<%= form_with model: @task, data: { turbo_frame: "_top" } do |f| %>
```

### Turbo Submission Events

Turbo fires events during the submission lifecycle that you can listen to for custom behavior.

```javascript
// app/javascript/controllers/form_events_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  // Listen for submission start
  submitStart(event) {
    const { formSubmission } = event.detail
    console.log("Submitting to:", formSubmission.fetchRequest.url)

    // Add a loading class to the form
    this.element.classList.add("submitting")
  }

  // Listen for submission end
  submitEnd(event) {
    const { success } = event.detail

    this.element.classList.remove("submitting")

    if (success) {
      // Form submitted successfully (2xx or 3xx response)
      console.log("Form submitted successfully")
    } else {
      // Form submission failed (4xx or 5xx response)
      console.log("Form submission failed")
    }
  }
}
```

```erb
<%= form_with model: @task,
      data: {
        controller: "form-events",
        action: "turbo:submit-start->form-events#submitStart turbo:submit-end->form-events#submitEnd"
      } do |f| %>
```

### Disabling Submit Buttons During Submission

Turbo automatically disables submit buttons during form submission to prevent double-clicks. The button receives a `disabled` attribute when the form starts submitting and the attribute is removed when the submission completes.

This is automatic -- you do not need to add any JavaScript or data attributes for basic disable behavior.

### data-turbo-submits-with for Button Text

To change the button's text during submission (e.g., "Save" becomes "Saving..."), use the `data-turbo-submits-with` attribute:

```erb
<%# The button text changes during submission %>
<%= f.submit "Save Task",
      data: { turbo_submits_with: "Saving..." } %>

<%# More examples %>
<%= f.submit "Create",
      data: { turbo_submits_with: "Creating..." } %>

<%= f.submit "Delete",
      data: { turbo_submits_with: "Deleting..." },
      class: "bg-red-600 text-white" %>

<%# For button tags (not f.submit) %>
<button type="submit"
        data-turbo-submits-with="Sending...">
  Send Message
</button>
```

The original text is automatically restored after the submission completes (success or failure).

### Handling Non-GET Form Submissions in Frames

When a form inside a Turbo Frame submits with POST/PATCH/DELETE:

1. The form submits to the server.
2. If the server returns 422, the frame content is replaced with the response body's matching frame.
3. If the server returns a redirect (303), Turbo follows the redirect. The redirected page's response must contain a matching frame, or Turbo falls back to full-page navigation.

```ruby
def create
  @task = @project.tasks.build(task_params)
  if @task.save
    # Option A: Redirect to a page that contains the matching frame
    redirect_to project_tasks_path(@project), status: :see_other

    # Option B: Respond with Turbo Streams for more control
    # respond_to do |format|
    #   format.turbo_stream
    #   format.html { redirect_to project_tasks_path(@project), status: :see_other }
    # end
  else
    render :new, status: :unprocessable_entity
  end
end
```

## Common Pitfalls

### Returning 200 for Validation Errors

```ruby
# BAD -- Turbo treats 200 as a success and replaces the page
render :new

# GOOD -- Turbo re-renders the form in place
render :new, status: :unprocessable_entity
```

This is the number one Turbo form bug. Without `status: :unprocessable_entity`, Turbo interprets the 200 response as a new page and performs a full page replacement, losing the form context.

### Using 302 Instead of 303

```ruby
# BAD -- 302 can cause the browser to re-POST on redirect
redirect_to tasks_path

# GOOD -- 303 guarantees a GET request for the redirect
redirect_to tasks_path, status: :see_other
```

Rails defaults `redirect_to` to 302, but Turbo expects 303 for non-GET form submissions. Without `status: :see_other`, some browsers may re-POST the form data to the redirect URL.

### Forgetting the Frame in the Redirect Response

If a form inside a frame submits and the redirect response does not contain a matching frame, Turbo will log an error and fall back to full-page navigation. Either ensure the redirect target contains the matching frame, or use `data-turbo-frame="_top"` on the form to opt out of frame-scoped navigation.

## Pattern Card

### GOOD: Proper 422/303 Lifecycle with Button State

```erb
<%= form_with model: @task do |f| %>
  <% if @task.errors.any? %>
    <div class="bg-red-50 p-4 rounded-md mb-4">
      <ul>
        <% @task.errors.full_messages.each do |msg| %>
          <li class="text-sm text-red-700"><%= msg %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <%= f.text_field :title %>
  <%= f.text_area :description %>
  <%= f.submit "Save", data: { turbo_submits_with: "Saving..." } %>
<% end %>
```

```ruby
def create
  @task = Task.new(task_params)
  if @task.save
    redirect_to tasks_path, status: :see_other    # 303 -> Turbo follows redirect
  else
    render :new, status: :unprocessable_entity      # 422 -> Turbo re-renders form
  end
end
```

This approach uses the correct 422/303 status codes that Turbo requires, `data-turbo-submits-with` for automatic button state management, server-rendered error messages that appear naturally on 422 re-render, and standard Rails form patterns with no custom JavaScript.

### BAD: 200 Status for Errors Breaking Turbo

```ruby
def create
  @task = Task.new(task_params)
  if @task.save
    # 302 instead of 303 -- may cause re-POST on redirect
    redirect_to tasks_path
  else
    # 200 instead of 422 -- Turbo replaces the entire page instead of re-rendering the form
    render :new
  end
end
```

```erb
<%# Manual button disabling instead of using Turbo's built-in support %>
<%= f.submit "Save", onclick: "this.disabled=true; this.form.submit();" %>
```

This approach returns 200 for validation errors which causes Turbo to perform a full page replacement instead of an in-place form re-render. The manual button disable uses `this.form.submit()` which bypasses Turbo entirely. The 302 redirect may cause browsers to replay the POST request.
