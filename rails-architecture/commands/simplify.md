---
description: Plan incremental refactoring toward DHH-style Rails architecture
disable-model-invocation: true
---

# /rails:simplify

Plan incremental refactoring toward DHH-style Rails architecture. Each step is independently valuable and reversible.

## Usage

```
/rails:simplify                                    # Plan overall simplification
/rails:simplify [goal]                             # Plan toward a specific goal
/rails:simplify "remove service objects"           # Target service layer
/rails:simplify "enrich Order model"               # Target specific model
/rails:simplify "convert to REST resources"        # Target routing
/rails:simplify "adopt Rails 8 defaults"           # Target infrastructure
```

## Process

### 1. Identify Current State

Scan the codebase to understand what exists:
- Run `/rails:analyze` (or equivalent scan) to build the full picture
- If a specific goal is provided, focus the scan on relevant areas
- Document the current architecture with counts and examples

### 2. Map to Target Patterns

For each area identified, determine the target DHH-style pattern:

| Current Pattern | Target Pattern | Reference |
|----------------|---------------|-----------|
| Service object (single model) | Rich model method | [fat-models-thin-controllers.md](../skills/rails-architecture/references/fat-models-thin-controllers.md) |
| Boolean state column | Dedicated state record + concern | [state-as-records.md](../skills/rails-architecture/references/state-as-records.md) |
| Custom controller action | Nested REST resource controller | [rest-resource-mapping.md](../skills/rails-architecture/references/rest-resource-mapping.md) |
| Anemic model | Rich model with concerns | [concerns-composition.md](../skills/rails-architecture/references/concerns-composition.md) |
| Fat controller | Thin controller + rich model | [fat-models-thin-controllers.md](../skills/rails-architecture/references/fat-models-thin-controllers.md) |
| Job with business logic | Shallow job + model method | [current-attributes-jobs.md](../skills/rails-architecture/references/current-attributes-jobs.md) |
| Sidekiq/Redis | Solid Queue/Cache/Cable | [rails-8-defaults.md](../skills/rails-architecture/references/rails-8-defaults.md) |
| RSpec + FactoryBot | Minitest + Fixtures | [minitest-fixtures.md](../skills/rails-architecture/references/minitest-fixtures.md) |

### 3. Create Migration Plan

Organize changes into incremental steps, ordered by:
1. **Risk:** Low-risk changes first
2. **Impact:** High-value changes prioritized
3. **Independence:** Each step works on its own
4. **Reversibility:** Easy to revert if needed

### 4. Prioritize by Impact

Assign effort and impact to each step:

| Effort | Impact | Priority |
|--------|--------|----------|
| Low | High | Do first |
| Low | Low | Do when convenient |
| High | High | Plan carefully |
| High | Low | Skip or defer |

## Simplification Strategies

### Strategy: Remove Service Objects

**Phase 1: Delete obvious thin wrappers (Low risk, High impact)**
- Services with < 15 lines
- Services called from exactly one place
- Services that wrap a single `model.update!` or `model.create!`

For each service:
1. Read the service and its one caller
2. Move the logic to the model as a method
3. Update the caller to use the model method
4. Delete the service file
5. Run tests

**Phase 2: Move domain logic to models (Medium risk, High impact)**
- Services containing business rules (calculations, validations, state transitions)
- Identify which model owns the data the rules operate on
- Move rules to that model as methods or concerns

**Phase 3: Convert multi-model services to POROs (Low risk, Medium impact)**
- Services that orchestrate multiple models
- Convert to plain Ruby objects with `ActiveModel::Model`
- Give them domain names: `Signup`, not `SignupService`

### Strategy: Convert Boolean State to Records

For each boolean state column:

1. **Create the state record model and migration**
   ```ruby
   # db/migrate/xxx_create_closures.rb
   create_table :closures do |t|
     t.references :card, null: false, foreign_key: true, index: { unique: true }
     t.references :user, null: false, foreign_key: true
     t.timestamps
   end
   ```

2. **Create the concern**
   ```ruby
   module Card::Closeable
     extend ActiveSupport::Concern
     included do
       has_one :closure, dependent: :destroy
       scope :closed, -> { joins(:closure) }
       scope :open, -> { where.missing(:closure) }
     end
     # ...
   end
   ```

3. **Migrate existing data**
   ```ruby
   Card.where(closed: true).find_each do |card|
     Closure.create!(card: card, user_id: card.closed_by_id, created_at: card.closed_at)
   end
   ```

4. **Update all callers** (controllers, views, tests)

5. **Remove the boolean column** (separate deploy)

### Strategy: Convert Custom Actions to REST Resources

For each custom action:

1. **Identify the noun** (close -> closure, archive -> archival)
2. **Create the nested resource controller**
3. **Update routes** from `member { post :close }` to `resource :closure`
4. **Move the action body** to `create` or `destroy` on the new controller
5. **Update links/forms** in views

### Strategy: Slim Fat Controllers

For each fat controller action:

1. **Identify business logic** in the action body
2. **Determine which model owns it**
3. **Create a model method** with an intention-revealing name
4. **Replace the controller logic** with a single model method call
5. **The controller should be 3-8 lines**

### Strategy: Adopt Rails 8 Infrastructure

Migrate incrementally, one component at a time:

1. **Solid Cache** (lowest risk: cache misses are harmless)
2. **Solid Cable** (medium risk: test WebSocket behavior)
3. **Solid Queue** (higher risk: test job processing thoroughly)
4. **Propshaft** (medium risk: test all assets load correctly)

## Output Format

```markdown
## Rails Simplification Plan

### Goal
[The specified goal, or "Overall simplification toward DHH-style Rails"]

### Current State
[Brief description of current architecture based on scan]

### Proposed Changes

#### Step 1: [Quick Win] (Low Risk, High Impact)

**Action:** Delete `ProcessOrderService` and move logic to `Order#process`
**Files Changed:**
- `app/services/process_order_service.rb` (DELETE)
- `app/models/order.rb` (ADD `process` method)
- `app/controllers/orders_controller.rb` (UPDATE to call `@order.process`)
- `test/models/order_test.rb` (ADD test for `process`)
**Effort:** ~30 minutes
**Risk:** Low (single caller, straightforward move)

```ruby
# Before: Controller calls service
class OrdersController < ApplicationController
  def create
    @order = ProcessOrderService.new(order_params).call
    redirect_to @order
  end
end

# After: Controller calls model
class OrdersController < ApplicationController
  def create
    @order = current_account.orders.create!(order_params)
    @order.process
    redirect_to @order
  end
end

# New model method
class Order < ApplicationRecord
  def process
    transaction do
      calculate_totals
      update!(status: :processing)
      send_confirmation_later
    end
  end
end
```

**Verification:** Run `bin/rails test test/models/order_test.rb test/controllers/orders_controller_test.rb`

---

#### Step 2: [Medium Win] (Low Risk, Medium Impact)

**Action:** [Description]
**Files Changed:** [List]
**Effort:** ~2 hours
**Risk:** Low-Medium

[Code examples]

**Verification:** [Test commands]

---

#### Step 3: [Larger Change] (Medium Risk, High Impact)

**Action:** [Description]
**Files Changed:** [List]
**Effort:** ~1 day
**Risk:** Medium

[Code examples]

**Verification:** [Test commands]

### Rollback Plan

Each step is independently reversible:
- Step 1: Restore `process_order_service.rb` from git, revert model and controller changes
- Step 2: [Specific rollback instructions]
- Step 3: [Specific rollback instructions]

### Success Criteria

After completing all steps:
- [ ] All tests pass
- [ ] No service objects for single-model operations remain
- [ ] All controller actions are 3-8 lines
- [ ] All models have intention-revealing APIs
- [ ] `rails routes` shows only standard CRUD actions

### Estimated Impact
- **LOC removed:** ~N lines
- **Files deleted:** N service files
- **Models enriched:** N models with new methods
- **Controllers slimmed:** N controllers
```

## Stop Points

Each step is independently valuable. Stop after any step if:
- The immediate goal is achieved
- Risk becomes unacceptable for the next step
- The team wants to review progress before continuing
- Tests are failing and the cause is unclear

**The golden rule:** Leave the codebase better than you found it, even if you only complete Step 1.
