---
title: Minitest with Fixtures
category: testing
updated: 2026-02-17
---

# Minitest with Fixtures

Ship what Rails ships. Minitest is fast, simple, and sufficient. Fixtures provide deterministic test data that encourages thinking about your domain as a coherent whole.

## Table of Contents

- [Overview](#overview)
- [Implementation](#implementation)
  - [Fixture File Structure](#fixture-file-structure)
  - [Fixture Relationships](#fixture-relationships)
  - [Test Organization](#test-organization)
  - [Assertion Patterns](#assertion-patterns)
  - [Testing Models](#testing-models)
  - [Testing Controllers](#testing-controllers)
  - [System Tests](#system-tests)
- [Pattern Card](#pattern-card)

## Overview

DHH and 37signals use Minitest with fixtures exclusively. No RSpec. No FactoryBot. This is a deliberate choice with specific reasoning:

**Why Minitest over RSpec:**
- Minitest ships with Rails. Zero additional dependencies.
- Plain Ruby classes and methods. No DSL to learn.
- `assert` and `refute` are simple and unambiguous.
- No `let`, `subject`, `shared_examples`, or matcher gems to create indirection.
- Tests read as straightforward Ruby code.

**Why Fixtures over FactoryBot:**
- Fixtures are loaded once per test suite, not rebuilt per test. They are fast.
- Fixtures force you to think about your test data as a **coherent whole**: a realistic snapshot of your database with proper relationships.
- With factories, each test builds its own isolated island of data, often with subtle inconsistencies.
- Fixtures encourage **named references** (`cards(:open_card)`) that make tests read like documentation.
- When you add a new required column, you update fixtures once. With factories, you update every test that builds that model.

**The Fizzy approach:** 37signals' Campfire codebase uses Minitest exclusively. Tests are fast, deterministic, and read as plain Ruby.

## Implementation

### Fixture File Structure

Fixtures live in `test/fixtures/` and are organized by table name:

```
test/fixtures/
  users.yml
  accounts.yml
  boards.yml
  cards.yml
  closures.yml
  comments.yml
```

Each fixture file defines named records:

```yaml
# test/fixtures/users.yml
admin:
  name: Alice Admin
  email: alice@example.com
  role: admin

member:
  name: Bob Member
  email: bob@example.com
  role: member

viewer:
  name: Carol Viewer
  email: carol@example.com
  role: viewer
```

```yaml
# test/fixtures/accounts.yml
acme:
  name: Acme Corp
  owner: admin

startup:
  name: Startup Inc
  owner: member
```

### Fixture Relationships

Reference other fixtures by name. Rails resolves the foreign keys automatically:

```yaml
# test/fixtures/boards.yml
engineering:
  name: Engineering
  account: acme
  creator: admin

marketing:
  name: Marketing
  account: acme
  creator: member
```

```yaml
# test/fixtures/cards.yml
open_card:
  title: Open Task
  board: engineering
  creator: admin

closed_card:
  title: Completed Task
  board: engineering
  creator: member
```

```yaml
# test/fixtures/closures.yml
closed_card_closure:
  card: closed_card
  user: admin
```

**Tip:** Name fixtures descriptively so tests read well: `cards(:open_card)`, `users(:admin)`, `boards(:engineering)`.

### Test Organization

Organize tests to mirror your app structure:

```
test/
  models/
    card_test.rb
    card/
      closeable_test.rb
      publishable_test.rb
  controllers/
    cards_controller_test.rb
    cards/
      closures_controller_test.rb
  system/
    card_flows_test.rb
  test_helper.rb
```

### Assertion Patterns

Minitest provides simple, powerful assertions:

```ruby
class CardTest < ActiveSupport::TestCase
  # State assertions
  test "new card is open by default" do
    card = cards(:open_card)
    assert card.open?
    refute card.closed?
  end

  # Change assertions
  test "closing a card creates a closure record" do
    card = cards(:open_card)
    assert_difference "Closure.count", 1 do
      card.close(user: users(:admin))
    end
  end

  # No-change assertions
  test "closing an already closed card does nothing" do
    card = cards(:closed_card)
    assert_no_difference "Closure.count" do
      card.close(user: users(:admin))
    end
  end

  # Exception assertions
  test "cannot create card without title" do
    assert_raises ActiveRecord::RecordInvalid do
      Card.create!(title: nil, board: boards(:engineering))
    end
  end

  # Collection assertions
  test "open scope excludes closed cards" do
    open_cards = Card.open
    assert_includes open_cards, cards(:open_card)
    refute_includes open_cards, cards(:closed_card)
  end

  # Numeric assertions
  test "order calculates correct total" do
    order = orders(:pending)
    order.calculate_totals
    assert_in_delta 99.99, order.total, 0.01
  end
end
```

### Testing Models

Test model behavior, not implementation:

```ruby
class Card::CloseableTest < ActiveSupport::TestCase
  setup do
    @card = cards(:open_card)
    @user = users(:admin)
  end

  test "close creates closure with user" do
    @card.close(user: @user)

    assert @card.closed?
    assert_equal @user, @card.closure.user
  end

  test "close is idempotent" do
    @card.close(user: @user)
    assert_no_difference "Closure.count" do
      @card.close(user: @user)
    end
  end

  test "reopen destroys closure" do
    @card.close(user: @user)
    @card.reopen(user: @user)

    refute @card.closed?
    assert_nil @card.reload.closure
  end

  test "closed scope returns only closed cards" do
    assert_includes Card.closed, cards(:closed_card)
    refute_includes Card.closed, cards(:open_card)
  end

  test "open scope returns only open cards" do
    assert_includes Card.open, cards(:open_card)
    refute_includes Card.open, cards(:closed_card)
  end
end
```

### Testing Controllers

Test HTTP behavior, not internal implementation:

```ruby
class Cards::ClosuresControllerTest < ActionDispatch::IntegrationTest
  setup do
    @card = cards(:open_card)
    sign_in users(:admin)
  end

  test "create closes the card" do
    post card_closure_path(@card)

    assert_redirected_to @card.board
    assert @card.reload.closed?
  end

  test "destroy reopens the card" do
    @card.close(user: users(:admin))

    delete card_closure_path(@card)

    assert_redirected_to @card.board
    refute @card.reload.closed?
  end

  test "create responds with turbo stream" do
    post card_closure_path(@card), as: :turbo_stream

    assert_response :success
    assert_equal "text/vnd.turbo-stream.html", response.media_type
  end
end
```

### System Tests

Test full user flows in the browser:

```ruby
class CardFlowsTest < ApplicationSystemTestCase
  setup do
    sign_in users(:admin)
  end

  test "closing a card from the board view" do
    visit board_path(boards(:engineering))

    within "#card_#{cards(:open_card).id}" do
      click_on "Close"
    end

    assert_text "Card closed"
    refute_selector "#card_#{cards(:open_card).id}"
  end

  test "creating a new card" do
    visit board_path(boards(:engineering))

    click_on "New Card"
    fill_in "Title", with: "My New Card"
    click_on "Create Card"

    assert_text "My New Card"
  end
end
```

## Pattern Card

### GOOD: Fixture-Based Test with Clear Assertions

```ruby
# test/fixtures/cards.yml
open_card:
  title: Ship feature
  board: engineering
  creator: admin

closed_card:
  title: Done task
  board: engineering
  creator: member

# test/fixtures/closures.yml
closed_card_closure:
  card: closed_card
  user: admin

# test/models/card/closeable_test.rb
class Card::CloseableTest < ActiveSupport::TestCase
  test "close creates closure and tracks event" do
    card = cards(:open_card)

    assert_difference -> { Closure.count } => 1, -> { Event.count } => 1 do
      card.close(user: users(:admin))
    end

    assert card.closed?
    assert_equal users(:admin), card.closure.user
  end

  test "open scope excludes closed cards" do
    results = Card.open
    assert_includes results, cards(:open_card)
    refute_includes results, cards(:closed_card)
  end
end
```

**Why this is good:**
- Fixtures define a coherent dataset: an open card, a closed card with its closure
- Test references fixtures by name: `cards(:open_card)` reads like documentation
- `assert_difference` is concise and verifies side effects
- Plain Ruby: no DSL, no `let` blocks, no `subject`, no magic
- Fast: fixtures are loaded once, not rebuilt per test
- When you add a required column to cards, you update `cards.yml` once

### BAD: Factory-Based Test with Complex Setup

```ruby
# spec/factories/cards.rb
FactoryBot.define do
  factory :card do
    sequence(:title) { |n| "Card #{n}" }
    association :board
    association :creator, factory: :user

    trait :closed do
      after(:create) do |card|
        create(:closure, card: card, user: card.creator)
      end
    end

    trait :with_comments do
      after(:create) do |card|
        create_list(:comment, 3, card: card)
      end
    end
  end
end

# spec/models/card/closeable_spec.rb
RSpec.describe Card::Closeable do
  let(:user) { create(:user) }
  let(:board) { create(:board, creator: user) }
  let(:card) { create(:card, board: board, creator: user) }

  describe "#close" do
    context "when card is open" do
      it "creates a closure record" do
        expect { card.close(user: user) }
          .to change(Closure, :count).by(1)
          .and change(Event, :count).by(1)
      end

      it "marks the card as closed" do
        card.close(user: user)
        expect(card).to be_closed
        expect(card.closure.user).to eq(user)
      end
    end

    context "when card is already closed" do
      let(:card) { create(:card, :closed, board: board) }

      it "does not create another closure" do
        expect { card.close(user: user) }
          .not_to change(Closure, :count)
      end
    end
  end
end
```

**Why this is bad:**
- Every test creates its own isolated data: `create(:user)`, `create(:board)`, `create(:card)`
- Factory traits hide complexity: `:closed` trait runs an `after(:create)` callback
- `let` blocks create lazy-evaluated indirection: you must trace `let` chains to understand the test
- Slow: each test builds multiple database records from scratch
- When you add a required column, you update the factory AND potentially every test that uses traits
- DSL overhead: `describe`, `context`, `let`, `it`, `expect(...).to` vs plain `test` and `assert`
- The test structure mirrors RSpec convention, not the domain
