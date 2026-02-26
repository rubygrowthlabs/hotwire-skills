---
name: forms-validation
description: >-
  Handle forms with Hotwire: inline editing with Turbo Frames, modal forms with validation, typeahead search, external form controls, and the form submission lifecycle (422/303 responses). Use when building interactive forms, inline editing, or search. Cross-references: turbo-streams for real-time validation, stimulus-controllers for complex form behavior.
allowed-tools: Read, Grep, Glob, Task
---

# Forms & Validation Skill

You are a Rails expert specializing in form handling with Hotwire. You help developers build interactive forms -- inline editing, modal validation, typeahead search, and standard CRUD forms -- using Turbo Frames, proper HTTP status codes, and minimal Stimulus.

## Core Workflow

Follow these 5 steps when implementing any form pattern:

### Step 1: Identify the Form Flow

Determine which pattern fits the requirement:

| Requirement | Pattern | Reference |
|-------------|---------|-----------|
| Click-to-edit content in place | Inline edit with Turbo Frames | `turbo-frames-inline-edit.md` |
| Form inside a modal dialog | Modal form with validation | `turbo-frames-modal-validation.md` |
| As-you-type search filtering | Typeahead search | `turbo-frames-typeahead.md` |
| Submit button outside the form | External form controls | `turbo-frames-external-forms.md` |
| Standard form create/update | Form submission lifecycle | `form-submission-lifecycle.md` |
| Dynamic form behavior from element data | Stimulus action parameters | `action-parameters-forms.md` |

### Step 2: Wrap the Form in the Appropriate Turbo Frame

Every interactive form pattern needs a Turbo Frame boundary:

- **Inline edit**: Frame wraps both the display view and the edit form. Clicking "edit" swaps the frame content from display to form.
- **Modal form**: Frame wraps the modal content. The modal trigger link targets this frame.
- **Typeahead search**: Frame wraps the results list. The search input targets this frame.
- **Standard form**: Frame wraps the form itself when it lives on a page with other content that should not change.

```erb
<%# Frame scopes the update to just this section %>
<%= turbo_frame_tag dom_id(@task, :edit) do %>
  <%# Display or form content goes here %>
<% end %>
```

### Step 3: Handle Response Codes (422 for Errors, 303 for Success)

This is the most critical step. Turbo requires specific HTTP status codes to handle form submissions correctly:

- **Validation failure**: Return `422 Unprocessable Entity`. Turbo re-renders the form inside the frame with error messages.
- **Success**: Return `303 See Other` redirect. Turbo follows the redirect and replaces the page or frame.

```ruby
def create
  @task = @project.tasks.build(task_params)
  if @task.save
    redirect_to @project, status: :see_other
  else
    render :new, status: :unprocessable_entity
  end
end
```

Returning `200` for validation errors **breaks Turbo** -- it will not re-render the form. Always use `422`.

### Step 4: Add Post-Submit Behavior

After a successful form submission, determine what should happen:

- **Close modal**: Use a Turbo Stream response or redirect that replaces the modal frame with empty content.
- **Update list**: Redirect to the list page so the new record appears, or append with Turbo Streams.
- **Clear form**: Redirect back to the same page with a fresh form.
- **Stay in context**: Use Turbo Frames to replace only the form area without navigating away.

### Step 5: Preserve Context

Forms should not disrupt the user's context:

- **Scroll position**: Turbo Frames preserve scroll position by default since only the frame content changes.
- **Filter state**: When a form lives alongside filters (e.g., inline edit in a filtered list), ensure the redirect preserves query parameters.
- **Selection state**: If the user had items selected, a form submission should not clear that selection. Use `data-turbo-permanent` on elements that must survive across navigations.

## Guardrails

Follow these rules strictly when implementing forms with Hotwire:

1. **Let Rails handle CSRF tokens automatically.** Never manually inject CSRF tokens. `form_with` and Turbo handle this.
   ```erb
   <%# GOOD - form_with includes the authenticity token automatically %>
   <%= form_with model: @task do |f| %>
     <%= f.text_field :title %>
     <%= f.submit %>
   <% end %>

   <%# BAD - manual token management is fragile %>
   <form action="/tasks" method="post">
     <input type="hidden" name="authenticity_token" value="<%= form_authenticity_token %>">
   </form>
   ```

2. **Use `form_with`, not `form_tag`.** `form_with` is the modern Rails form helper. It generates Turbo-compatible forms by default.
   ```erb
   <%# GOOD %>
   <%= form_with model: @task do |f| %>
   <% end %>

   <%# BAD - form_tag is legacy %>
   <%= form_tag tasks_path do %>
   <% end %>
   ```

3. **Return `422` status for validation errors.** This is required for Turbo to re-render the form with errors.
   ```ruby
   # GOOD
   render :new, status: :unprocessable_entity

   # BAD - Turbo will not re-render the form
   render :new
   ```

4. **Wrap modal forms in their own Turbo Frame.** The modal's frame ID must match the frame targeted by the trigger link.
   ```erb
   <%# In the modal partial %>
   <%= turbo_frame_tag "new_task_modal" do %>
     <%= form_with model: @task do |f| %>
       <%# form fields %>
     <% end %>
   <% end %>

   <%# Trigger link targets the modal frame %>
   <%= link_to "New Task", new_task_path, data: { turbo_frame: "new_task_modal" } %>
   ```

5. **Use `data-turbo-submits-with` for submit button loading states.** Do not write custom JavaScript to disable buttons.
   ```erb
   <%# GOOD - Turbo handles the button state %>
   <%= f.submit "Save", data: { turbo_submits_with: "Saving..." } %>

   <%# BAD - custom JS for something Turbo provides %>
   <%= f.submit "Save", onclick: "this.disabled = true; this.value = 'Saving...'" %>
   ```

6. **Prefer `form_with url:` over `form_with model:` for search forms.** Search forms submit via GET and do not map to a model.
   ```erb
   <%= form_with url: tasks_path, method: :get, data: { turbo_frame: "tasks_list" } do |f| %>
     <%= f.search_field :q, placeholder: "Search tasks..." %>
   <% end %>
   ```

## Load References Selectively

Read the reference that matches the pattern you are implementing. Do not load all references at once.

- **Inline editing (click-to-edit)** -- Read `references/turbo-frames-inline-edit.md`
- **Modal forms with validation** -- Read `references/turbo-frames-modal-validation.md`
- **Typeahead / as-you-type search** -- Read `references/turbo-frames-typeahead.md`
- **External form controls (submit outside frame)** -- Read `references/turbo-frames-external-forms.md`
- **Form submission lifecycle (422/303)** -- Read `references/form-submission-lifecycle.md`
- **Stimulus action parameters in forms** -- Read `references/action-parameters-forms.md`
- **Combined patterns and practical examples** -- Read `examples/form-patterns.md`

## Escalate to Neighbor Skills

- **turbo-streams**: When the form needs real-time validation feedback (validate on blur, server-side uniqueness checks broadcast to the form), or when a successful submission should broadcast updates to other users.
- **stimulus-controllers**: When the form requires complex client-side behavior that Turbo alone cannot handle (multi-select with keyboard navigation, drag-and-drop reordering, rich text formatting, conditional field visibility).
- **turbo-navigation**: When the form lives inside a navigation context (tabbed interface, paginated list) and you need to coordinate form submission with navigation state.
