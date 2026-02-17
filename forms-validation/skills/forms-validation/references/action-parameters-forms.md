---
title: "Stimulus Action Parameters in Forms"
categories:
  - stimulus
  - forms
  - action-parameters
tags:
  - stimulus
  - action-params
  - dynamic-forms
  - data-params
  - conditional-fields
description: >-
  Passing data from form elements to Stimulus actions via params. Covers
  data-*-param attributes on form controls, dynamic form behavior like
  conditional field visibility, and element-specific actions without
  manual data attribute parsing.
---

# Stimulus Action Parameters in Forms

## Table of Contents

- [Overview](#overview)
- [How Action Parameters Work](#how-action-parameters-work)
- [Implementation](#implementation)
  - [Basic Action Parameters on Form Elements](#basic-action-parameters-on-form-elements)
  - [Conditional Field Visibility](#conditional-field-visibility)
  - [Dynamic Form Behavior from Select Options](#dynamic-form-behavior-from-select-options)
  - [Per-Button Actions with Params](#per-button-actions-with-params)
  - [Type Coercion](#type-coercion)
- [Action Parameters vs Data Attributes vs Values](#action-parameters-vs-data-attributes-vs-values)
- [Pattern Card](#pattern-card)

## Overview

Stimulus action parameters let you pass data from an HTML element to a Stimulus action method without parsing data attributes manually. When an element triggers an action, any `data-<controller>-<param-name>-param` attributes on that element are collected into a `params` object and passed to the action method via the event.

This is particularly useful in forms where different elements need to trigger the same action but with different configuration:

- A category select that shows different fields based on the selected value
- Multiple submit buttons that each set a different form action or value
- Toggle switches that pass their section identifier to a show/hide method

## How Action Parameters Work

```
HTML element with action + params        Stimulus action method
+--------------------------------------+  +---------------------------+
| <button                              |  | toggle(event) {           |
|   data-action="form#toggle"          |  |   const { section } =    |
|   data-form-section-param="billing"  |  |     event.params         |
| >                                    |->|   // section = "billing" |
|   Toggle Billing                     |  | }                        |
| </button>                            |  +---------------------------+
+--------------------------------------+
```

The param name is derived from the data attribute: `data-form-section-param` becomes `params.section` in the action method. The `form` part matches the controller name.

## Implementation

### Basic Action Parameters on Form Elements

Pass data from buttons, inputs, and other form elements to Stimulus actions:

```erb
<%# app/views/orders/new.html.erb %>
<%= form_with model: @order,
      data: { controller: "order-form" } do |f| %>

  <%# Each button passes its priority to the same action %>
  <div class="flex gap-2 mb-4">
    <% %w[standard express overnight].each do |speed| %>
      <button type="button"
              class="px-4 py-2 border rounded-md"
              data-action="order-form#selectShipping"
              data-order-form-speed-param="<%= speed %>"
              data-order-form-price-param="<%= shipping_price(speed) %>">
        <%= speed.capitalize %>
      </button>
    <% end %>
  </div>

  <%= f.hidden_field :shipping_speed,
        data: { order_form_target: "shippingInput" } %>
  <%= f.hidden_field :shipping_price,
        data: { order_form_target: "priceInput" } %>

  <p data-order-form-target="summary" class="text-sm text-gray-600"></p>
<% end %>
```

```javascript
// app/javascript/controllers/order_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["shippingInput", "priceInput", "summary"]

  selectShipping(event) {
    const { speed, price } = event.params

    this.shippingInputTarget.value = speed
    this.priceInputTarget.value = price
    this.summaryTarget.textContent = `${speed} shipping: $${price}`

    // Highlight the selected button
    this.element.querySelectorAll("[data-action='order-form#selectShipping']").forEach(btn => {
      btn.classList.toggle("bg-blue-100", btn === event.currentTarget)
      btn.classList.toggle("border-blue-500", btn === event.currentTarget)
    })
  }
}
```

### Conditional Field Visibility

Show or hide form sections based on a selection, using action params to identify which section to display:

```erb
<%# app/views/contacts/new.html.erb %>
<%= form_with model: @contact,
      data: { controller: "conditional-fields" } do |f| %>
  <div class="space-y-4">
    <div>
      <%= f.label :contact_type, class: "block text-sm font-medium" %>
      <%= f.select :contact_type,
            [["Individual", "individual"], ["Company", "company"]],
            {},
            {
              class: "w-full rounded-md border-gray-300",
              data: {
                action: "conditional-fields#toggle",
                conditional_fields_show_param: ""
              }
            } %>
    </div>

    <%# Individual-specific fields %>
    <div data-conditional-fields-target="section"
         data-section="individual"
         class="space-y-4">
      <%= f.label :first_name %>
      <%= f.text_field :first_name, class: "w-full rounded-md border-gray-300" %>

      <%= f.label :last_name %>
      <%= f.text_field :last_name, class: "w-full rounded-md border-gray-300" %>
    </div>

    <%# Company-specific fields %>
    <div data-conditional-fields-target="section"
         data-section="company"
         class="space-y-4 hidden">
      <%= f.label :company_name %>
      <%= f.text_field :company_name, class: "w-full rounded-md border-gray-300" %>

      <%= f.label :tax_id %>
      <%= f.text_field :tax_id, class: "w-full rounded-md border-gray-300" %>
    </div>
  </div>

  <%= f.submit "Save Contact",
        class: "mt-6 px-4 py-2 bg-blue-600 text-white rounded-md",
        data: { turbo_submits_with: "Saving..." } %>
<% end %>
```

```javascript
// app/javascript/controllers/conditional_fields_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["section"]

  toggle(event) {
    const selectedValue = event.currentTarget.value

    this.sectionTargets.forEach(section => {
      const sectionName = section.dataset.section
      const shouldShow = sectionName === selectedValue
      section.classList.toggle("hidden", !shouldShow)

      // Disable hidden inputs so they are not submitted
      section.querySelectorAll("input, select, textarea").forEach(input => {
        input.disabled = !shouldShow
      })
    })
  }
}
```

### Dynamic Form Behavior from Select Options

When different options require different form configurations, use action params to pass the configuration from the option:

```erb
<%= form_with model: @payment,
      data: { controller: "payment-form" } do |f| %>
  <div>
    <%= f.label :payment_method, class: "block text-sm font-medium" %>
    <div class="grid grid-cols-3 gap-3 mt-2">
      <% [
        { method: "credit_card", label: "Credit Card", fields: "card" },
        { method: "bank_transfer", label: "Bank Transfer", fields: "bank" },
        { method: "paypal", label: "PayPal", fields: "email" }
      ].each do |option| %>
        <button type="button"
                class="p-3 border rounded-lg text-center"
                data-action="payment-form#selectMethod"
                data-payment-form-method-param="<%= option[:method] %>"
                data-payment-form-fields-param="<%= option[:fields] %>">
          <%= option[:label] %>
        </button>
      <% end %>
    </div>
    <%= f.hidden_field :payment_method,
          data: { payment_form_target: "methodInput" } %>
  </div>

  <div data-payment-form-target="fieldsContainer" class="mt-4">
    <%# Fields are shown/hidden based on selection %>
    <div data-payment-form-target="cardFields" class="hidden space-y-3">
      <%= f.text_field :card_number, placeholder: "Card Number", class: "w-full rounded-md border-gray-300" %>
      <div class="grid grid-cols-2 gap-3">
        <%= f.text_field :expiry, placeholder: "MM/YY", class: "rounded-md border-gray-300" %>
        <%= f.text_field :cvv, placeholder: "CVV", class: "rounded-md border-gray-300" %>
      </div>
    </div>

    <div data-payment-form-target="bankFields" class="hidden space-y-3">
      <%= f.text_field :account_number, placeholder: "Account Number", class: "w-full rounded-md border-gray-300" %>
      <%= f.text_field :routing_number, placeholder: "Routing Number", class: "w-full rounded-md border-gray-300" %>
    </div>

    <div data-payment-form-target="emailFields" class="hidden space-y-3">
      <%= f.email_field :paypal_email, placeholder: "PayPal Email", class: "w-full rounded-md border-gray-300" %>
    </div>
  </div>

  <%= f.submit "Pay Now",
        class: "mt-6 w-full px-4 py-2 bg-green-600 text-white rounded-md",
        data: { turbo_submits_with: "Processing..." } %>
<% end %>
```

```javascript
// app/javascript/controllers/payment_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["methodInput", "cardFields", "bankFields", "emailFields"]

  selectMethod(event) {
    const { method, fields } = event.params

    // Update hidden input
    this.methodInputTarget.value = method

    // Show the appropriate fields section
    const fieldTargets = {
      card: this.cardFieldsTarget,
      bank: this.bankFieldsTarget,
      email: this.emailFieldsTarget
    }

    Object.entries(fieldTargets).forEach(([key, target]) => {
      const shouldShow = key === fields
      target.classList.toggle("hidden", !shouldShow)
      target.querySelectorAll("input").forEach(input => {
        input.disabled = !shouldShow
      })
    })

    // Highlight selected button
    this.element.querySelectorAll("[data-action='payment-form#selectMethod']").forEach(btn => {
      btn.classList.toggle("border-green-500", btn === event.currentTarget)
      btn.classList.toggle("bg-green-50", btn === event.currentTarget)
    })
  }
}
```

### Per-Button Actions with Params

When a form has multiple submit buttons that each do something different, use action params to differentiate:

```erb
<%= form_with model: @article,
      data: { controller: "article-form" } do |f| %>
  <%= f.text_field :title %>
  <%= f.text_area :body %>

  <div class="flex gap-3 mt-6">
    <button type="submit"
            data-action="article-form#submitWithStatus"
            data-article-form-status-param="draft"
            class="px-4 py-2 border rounded-md text-gray-700"
            data-turbo-submits-with="Saving Draft...">
      Save Draft
    </button>
    <button type="submit"
            data-action="article-form#submitWithStatus"
            data-article-form-status-param="published"
            class="px-4 py-2 bg-blue-600 text-white rounded-md"
            data-turbo-submits-with="Publishing...">
      Publish
    </button>
  </div>
<% end %>
```

```javascript
// app/javascript/controllers/article_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  submitWithStatus(event) {
    const { status } = event.params

    // Add the status as a hidden field before submission
    let hiddenField = this.element.querySelector("input[name='article[status]']")
    if (!hiddenField) {
      hiddenField = document.createElement("input")
      hiddenField.type = "hidden"
      hiddenField.name = "article[status]"
      this.element.appendChild(hiddenField)
    }
    hiddenField.value = status
  }
}
```

### Type Coercion

Stimulus automatically coerces action parameter values based on their content:

| Attribute Value | JavaScript Type | Result |
|----------------|-----------------|--------|
| `"hello"` | String | `"hello"` |
| `"42"` | Number | `42` |
| `"3.14"` | Number | `3.14` |
| `"true"` | Boolean | `true` |
| `"false"` | Boolean | `false` |
| `"{\"a\":1}"` | Object | `{ a: 1 }` |
| `"[1,2,3]"` | Array | `[1, 2, 3]` |

```erb
<button data-action="form#configure"
        data-form-count-param="5"
        data-form-enabled-param="true"
        data-form-config-param='{"max": 100}'>
  Configure
</button>
```

```javascript
configure(event) {
  const { count, enabled, config } = event.params
  // count = 5 (Number)
  // enabled = true (Boolean)
  // config = { max: 100 } (Object)
}
```

## Action Parameters vs Data Attributes vs Values

| Feature | When to Use | Scope |
|---------|------------|-------|
| Action Parameters | Data varies per element that triggers the action | Per-element |
| Stimulus Values | Configuration for the controller instance | Per-controller |
| Data Attributes (manual) | Avoid -- use action params or values instead | -- |

```erb
<%# Action params: different per button %>
<button data-action="form#submit" data-form-priority-param="high">High</button>
<button data-action="form#submit" data-form-priority-param="low">Low</button>

<%# Stimulus values: controller-level config %>
<form data-controller="form" data-form-max-length-value="500">

<%# BAD: manual data attributes %>
<button data-action="form#submit" data-priority="high">High</button>
```

## Pattern Card

### GOOD: Action Params for Form Behavior

```erb
<%= form_with model: @task,
      data: { controller: "task-form" } do |f| %>
  <%= f.text_field :title %>

  <%# Each button passes its intent via action params %>
  <button type="button"
          data-action="task-form#setPriority"
          data-task-form-level-param="high"
          data-task-form-color-param="red">
    High Priority
  </button>
  <button type="button"
          data-action="task-form#setPriority"
          data-task-form-level-param="low"
          data-task-form-color-param="gray">
    Low Priority
  </button>

  <%= f.submit "Save" %>
<% end %>
```

```javascript
// Clean, declarative parameter passing
setPriority(event) {
  const { level, color } = event.params
  this.priorityInputTarget.value = level
  this.indicatorTarget.style.color = color
}
```

This approach passes data declaratively from HTML to JavaScript, requires no manual `dataset` parsing, supports automatic type coercion, and keeps the controller methods generic and reusable.

### BAD: Data Attributes with Manual Parsing

```erb
<button data-action="task-form#setPriority"
        data-priority-level="high"
        data-priority-color="red">
  High Priority
</button>
```

```javascript
// Manual parsing -- verbose and error-prone
setPriority(event) {
  const level = event.currentTarget.dataset.priorityLevel
  const color = event.currentTarget.dataset.priorityColor
  // No type coercion -- "42" stays a string
  // Must remember to use currentTarget, not target
  // Attribute names are disconnected from the controller
}
```

This approach requires manual `dataset` access, provides no automatic type coercion, decouples the attribute naming from the controller name making refactoring error-prone, and requires remembering to use `event.currentTarget` instead of `event.target`.
