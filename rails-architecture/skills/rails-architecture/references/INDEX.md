# Reference Index

All reference articles for the `rails-architecture` skill. Each article is self-contained with an overview, implementation guidance, and pattern cards showing good vs. bad approaches.

## Articles

| # | File | Topic | Summary |
|---|------|-------|---------|
| 1 | [fat-models-thin-controllers.md](./fat-models-thin-controllers.md) | Rich Domain Models | Why models own business logic. When services are justified (multi-model orchestration only). Intention-revealing model APIs. |
| 2 | [concerns-composition.md](./concerns-composition.md) | Concerns as Composition | Adjective-named concerns for horizontal behavior sharing. `included` blocks, `class_methods`, focused single-capability concerns. |
| 3 | [state-as-records.md](./state-as-records.md) | State as Records | Tracking state with dedicated models instead of boolean columns. `Card.joins(:closure)` over `Card.where(closed: true)`. |
| 4 | [rest-resource-mapping.md](./rest-resource-mapping.md) | REST Resource Mapping | Every action maps to CRUD on a resource. Converting custom controller actions into new resource controllers. |
| 5 | [minitest-fixtures.md](./minitest-fixtures.md) | Minitest with Fixtures | Testing with what Rails ships. Fixture file structure, test organization, assertion patterns. Why not RSpec/FactoryBot. |
| 6 | [current-attributes-jobs.md](./current-attributes-jobs.md) | Current Attributes and Jobs | `Current.user` and `Current.account` for request context. Shallow job wrappers that delegate to model methods. |
| 7 | [rails-8-defaults.md](./rails-8-defaults.md) | Rails 8 Defaults | Solid Queue, Solid Cache, Solid Cable, Kamal, Propshaft. Database-backed infrastructure. Migration from Redis/Sidekiq. |
