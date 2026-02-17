# /rails:analyze

Scan an entire Rails codebase for simplification opportunities toward DHH-style architecture.

## Usage

```
/rails:analyze                      # Analyze entire codebase
/rails:analyze [path]               # Analyze specific directory
/rails:analyze --services           # Focus on service layer audit
/rails:analyze --models             # Focus on model health assessment
/rails:analyze --controllers        # Focus on controller thickness
/rails:analyze --infrastructure     # Focus on Rails 8 defaults adoption
```

## Process

### 1. Scan the app/ Directory

Build an inventory of the codebase by layer:

```
Scan these directories:
  app/controllers/    -> Count controllers, lines per action, custom actions
  app/models/         -> Count models, public methods, concerns included
  app/services/       -> Count services (these are suspects)
  app/jobs/           -> Count jobs, lines per perform method
  config/routes.rb    -> Count custom (non-CRUD) routes
  test/ or spec/      -> Detect test framework and data strategy
  Gemfile             -> Detect infrastructure dependencies
```

### 2. Identify Service Objects

Audit `app/services/` (if it exists):

For each service, determine:
- **How many models does it operate on?** (single-model = should be on the model)
- **How many callers does it have?** (single caller = likely unnecessary indirection)
- **Does it contain domain logic?** (business rules = belongs on the model)
- **What is its line count?** (< 15 lines = thin wrapper, likely unnecessary)

Categorize each service:
- **Delete immediately:** Single-model operation, thin wrapper, or domain logic that belongs on a model
- **Questionable:** Multi-model coordination that might be simplified
- **Justified:** External API integration, complex multi-model orchestration with no natural owner

### 3. Check Controller Sizes

For each controller action:
- Count lines (excluding blank/comment lines)
- Flag actions > 8 lines
- Detect business logic (calculations, domain conditionals, multi-step orchestration)
- Detect custom (non-CRUD) actions

### 4. Assess Model Richness

For each model:
- Count public instance methods (excluding Rails-generated)
- Detect anemic models (0-2 custom public methods)
- List included concerns
- Check for boolean state columns that should be records
- Look for `app/services/` files named after this model

### 5. Audit Infrastructure

Check for Rails 8 defaults adoption:
- **Queue adapter:** Solid Queue vs Sidekiq/Resque/DelayedJob
- **Cache store:** Solid Cache vs Redis/Memcached
- **Cable adapter:** Solid Cable vs Redis
- **Asset pipeline:** Propshaft vs Sprockets
- **Authentication:** Rails 8 generator vs Devise
- **Test framework:** Minitest vs RSpec
- **Test data:** Fixtures vs FactoryBot

### 6. Report Findings with Priority

Generate a comprehensive report organized by severity and effort.

## Output Format

```markdown
## Rails Architecture Analysis

### Codebase Overview

| Layer | Count | Health |
|-------|-------|--------|
| Controllers | 24 | 3 fat controllers detected |
| Models | 18 | 5 anemic models detected |
| Services | 12 | 8 unnecessary, 2 questionable, 2 justified |
| Jobs | 6 | 2 with business logic |
| Concerns | 8 | Well-composed |

### Service Layer Audit

**Total Services Found:** 12

**Delete These (single-model wrappers):**
| Service | Lines | Model | Recommendation |
|---------|-------|-------|----------------|
| `ProcessOrderService` | 25 | Order | Move to `Order#process` |
| `CreateUserService` | 15 | User | Delete, use controller + model |
| `UpdateProfileService` | 10 | User | Delete, use `User#update!` directly |
| `CalculateCartTotalService` | 20 | Cart | Move to `Cart#calculate_total` |
| `CloseCardService` | 8 | Card | Move to `Card#close` via Closeable concern |

**Questionable (review needed):**
| Service | Lines | Models Touched | Assessment |
|---------|-------|----------------|------------|
| `SignupService` | 45 | User, Account, Subscription | Multi-model, may be justified. Consider `Signup` PORO with ActiveModel::Model |
| `TransferOwnershipService` | 30 | Account, User, Board | Multi-model orchestration, review if simpler approach exists |

**Justified (keep):**
| Service | Lines | Reason |
|---------|-------|--------|
| `StripeWebhookHandler` | 60 | External API integration |
| `DataExportGenerator` | 80 | Complex cross-model report generation |

### Model Health

**Anemic Models (need enrichment):**
| Model | Custom Methods | Missing APIs |
|-------|---------------|--------------|
| `User` | 0 | `authenticate`, `can_perform?`, `promote` |
| `Order` | 1 | `process`, `complete`, `cancel`, `refund` |
| `Card` | 0 | `close`, `archive`, `transfer_to` |

**Rich Models (good):**
| Model | Custom Methods | Concerns |
|-------|---------------|----------|
| `Board` | 5 | Archivable, Publishable |
| `Comment` | 3 | Mentionable, Broadcastable |

**Boolean State Columns (should be records):**
| Model | Column | Recommended Record |
|-------|--------|-------------------|
| `Card.closed` | boolean | `Closure` model |
| `Card.archived` | boolean | `Archival` model |
| `Post.published` | boolean | `Publication` model |

### Controller Thickness

**Fat Controllers (> 8 lines per action):**
| Controller | Action | Lines | Problem |
|-----------|--------|-------|---------|
| `OrdersController#create` | create | 25 | Pricing logic in controller |
| `UsersController#update` | update | 18 | Profile validation in controller |
| `CardsController#close` | close | 12 | Custom action + business logic |

**Custom (Non-CRUD) Actions:**
| Controller | Action | Recommended Resource |
|-----------|--------|---------------------|
| `CardsController#close` | close | `Cards::ClosuresController#create` |
| `CardsController#archive` | archive | `Cards::ArchivalsController#create` |
| `PostsController#publish` | publish | `Posts::PublicationsController#create` |

### Infrastructure Audit

| Component | Current | Recommended | Effort |
|-----------|---------|-------------|--------|
| Queue | Sidekiq + Redis | Solid Queue | Medium |
| Cache | Redis | Solid Cache | Low |
| Cable | Redis | Solid Cable | Low |
| Assets | Sprockets | Propshaft | Medium |
| Auth | Devise | Rails 8 generator | High |
| Tests | RSpec | Minitest | High |
| Factories | FactoryBot | Fixtures | High |

### Recommendations

**Immediate Wins (< 1 hour each):**
1. Delete 5 unnecessary services, move logic to models (~200 LOC removed)
2. Slim 3 fat controllers by extracting logic to models
3. Switch cache store from Redis to Solid Cache

**Short-Term (1-5 hours each):**
1. Convert 3 boolean state columns to state records
2. Convert 3 custom controller actions to REST resources
3. Switch Action Cable from Redis to Solid Cable

**Medium-Term (1-3 days each):**
1. Migrate from Sidekiq to Solid Queue
2. Enrich 5 anemic models with business logic

**Long-Term (consider carefully):**
1. Migrate from RSpec to Minitest
2. Migrate from FactoryBot to Fixtures
3. Replace Devise with Rails 8 authentication

### Stats Summary
- **Services to remove:** 8
- **Models to enrich:** 5
- **Controllers to slim:** 3
- **Boolean columns to convert:** 3
- **Custom actions to convert:** 3
- **Estimated LOC reduction:** ~500 lines
```

## Automation Level

1. **Automatic:** Scan directories and count files
2. **Automatic:** Measure controller action sizes
3. **Automatic:** Detect service objects and their model dependencies
4. **Automatic:** Identify boolean state columns in migrations/schema
5. **Automatic:** Check Gemfile for infrastructure dependencies
6. **Judgment required:** Whether specific services are justified
7. **Judgment required:** Priority ordering based on team context
