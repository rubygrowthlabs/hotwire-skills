---
description: Review code against DHH-style Rails 8 architecture principles
disable-model-invocation: true
---

# /rails:review

Review code against DHH-style Rails 8 architecture principles.

## Usage

```
/rails:review                       # Review uncommitted changes
/rails:review --staged              # Review staged changes only
/rails:review [file_path]           # Review a specific file or directory
/rails:review --branch main         # Review all changes vs a branch
```

## Process

### 1. Identify Files to Review

Determine the scope based on invocation:

- **No arguments:** Run `git diff` and `git diff --cached` to find all changed files
- **`--staged`:** Run `git diff --cached` for staged changes only
- **`[file_path]`:** Review the specified file or all files in the specified directory
- **`--branch [branch]`:** Run `git diff [branch]...HEAD` for all changes since branching

### 2. Read References

Before reviewing, read the relevant references from the rails-architecture skill based on the file types found:

- Controllers found -> Read [fat-models-thin-controllers.md](../skills/rails-architecture/references/fat-models-thin-controllers.md) and [rest-resource-mapping.md](../skills/rails-architecture/references/rest-resource-mapping.md)
- Models found -> Read [fat-models-thin-controllers.md](../skills/rails-architecture/references/fat-models-thin-controllers.md) and [concerns-composition.md](../skills/rails-architecture/references/concerns-composition.md)
- Services found -> Read [fat-models-thin-controllers.md](../skills/rails-architecture/references/fat-models-thin-controllers.md) (services section)
- Migrations with boolean state columns -> Read [state-as-records.md](../skills/rails-architecture/references/state-as-records.md)
- Tests found -> Read [minitest-fixtures.md](../skills/rails-architecture/references/minitest-fixtures.md)
- Jobs found -> Read [current-attributes-jobs.md](../skills/rails-architecture/references/current-attributes-jobs.md)
- Infrastructure config -> Read [rails-8-defaults.md](../skills/rails-architecture/references/rails-8-defaults.md)

### 3. Detect Anti-Patterns

For each file, check against the review checklist below. Use the `rails-reviewer` agent for deep analysis when needed.

### 4. Assess Controller Thickness

For every controller action in the changed files:
- Count lines per action (target: 3-8 lines)
- Flag any business logic (calculations, domain conditionals)
- Verify only standard CRUD actions exist (index, show, new, create, edit, update, destroy)
- Check that params are parsed with strong parameters

### 5. Evaluate Model Richness

For every model in the changed files:
- List public methods -- are they intention-revealing?
- Check concern composition -- are concerns focused and adjective-named?
- Verify state tracking -- records or booleans?
- Look for domain logic that should be here but is elsewhere

### 6. Generate Report

Produce a structured report with severity levels, specific file locations, problematic code snippets, and concrete fix suggestions with code examples.

## Review Checklist

### Architecture (Critical)
- [ ] No service objects for single-model operations
- [ ] No business logic in services that belongs on models
- [ ] No anemic models (attributes + associations only, no behavior)
- [ ] No boolean columns tracking state that should be records
- [ ] No custom controller actions (should be separate REST resources)
- [ ] No `app/services/` files without clear multi-model justification

### Controller Health (Warning)
- [ ] Controllers are thin (3-8 lines per action)
- [ ] Controllers only handle HTTP concerns (params, response)
- [ ] Controllers call rich model methods directly
- [ ] No business logic (calculations, conditionals) in controllers
- [ ] Only standard CRUD actions in each controller

### Model Health (Warning)
- [ ] Models contain business logic as instance methods
- [ ] Models have intention-revealing APIs (`order.process`, not `ProcessOrderService.call`)
- [ ] Concerns are adjective-named and focused on one capability
- [ ] State tracked with dedicated records, not boolean columns
- [ ] `belongs_to` defaults use `Current` where appropriate

### Jobs (Warning)
- [ ] Jobs are shallow wrappers (3-5 lines, one method call in `perform`)
- [ ] Business logic lives on the model, not in the job
- [ ] `_later` / `_now` naming convention for async methods

### Testing (Suggestion)
- [ ] Tests use Minitest (not RSpec)
- [ ] Test data uses fixtures (not FactoryBot)
- [ ] Assertions are simple: `assert`, `refute`, `assert_difference`
- [ ] Tests reference fixtures by name: `cards(:open_card)`

### Infrastructure (Suggestion)
- [ ] Solid Queue for background jobs (not Sidekiq/Redis)
- [ ] Solid Cache for caching (not Redis/Memcached)
- [ ] Solid Cable for Action Cable (not Redis pub/sub)
- [ ] Propshaft for assets (not Sprockets)

### Style (Suggestion)
- [ ] Expanded conditionals preferred over guard clauses
- [ ] Method ordering: class methods -> public -> private
- [ ] No newline after `private` keyword, indent private methods
- [ ] `%i[ ]` for symbol arrays with spaces inside brackets
- [ ] Bang methods (`create!`, `update!`) for fail-fast

## Output Format

```markdown
## Rails Architecture Review

### Scope
[Description of what was reviewed: N files, which layers]

### Files Reviewed
- `path/to/file.rb` (layer classification)

### Issues

**CRITICAL: [Issue Title]**
Location: `file:line`
[Code snippet, problem description, fix with code example]

**WARNING: [Issue Title]**
Location: `file:line`
[Problem description, recommendation]

**SUGGESTION: [Issue Title]**
Location: `file:line`
[Suggestion with rationale]

### Summary

**Good:**
- [Positive observations]

**Needs Attention:**
1. CRITICAL: [Top priority]
2. WARNING: [Second priority]
3. SUGGESTION: [Nice to have]

**Priority:** [What to fix first and why]
```

## Severity Levels

### CRITICAL
Must fix before merge:
- Unnecessary service layer for single-model operations
- Business logic in services instead of models
- Anemic models with all logic extracted
- Boolean state columns that should be records
- Non-REST custom controller actions

### WARNING
Should fix or explicitly acknowledge:
- Fat controllers with business logic
- Jobs containing business logic
- Missing intention-revealing model APIs
- Unfocused or noun-named concerns

### SUGGESTION
Consider for code quality:
- RSpec -> Minitest
- FactoryBot -> Fixtures
- Redis -> Solid Queue/Cache/Cable
- Style preferences
