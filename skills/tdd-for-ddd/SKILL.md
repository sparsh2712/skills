---
name: tdd-for-ddd
description: Test-Driven Development workflow for Domain-Driven Design backends in Python and FastAPI. Drives feature development through the RED-GREEN-REFACTOR cycle using fake repositories and fake UoW — no mocks allowed. Use this skill whenever the user wants to add a feature, fix a bug, or implement any behavior in a DDD-structured Python backend. Trigger on signals like writing tests first, adding domain behavior, implementing a use case, building a new aggregate, adding service layer logic, or any request where the user says "let's TDD this", "write tests for", "test-driven", "add a feature to the backend", or "implement this behavior". Also trigger when reviewing or writing tests for code that follows the repository/UoW/service-layer pattern, even if the user doesn't explicitly say "TDD".
---

# Test-Driven Development for Domain-Driven Design

## Core Philosophy

**Tests are the source of truth.** Every behavior in the system is proven by a test that was written first, watched fail, and then made to pass with minimal code. If a behavior has no failing test, it doesn't get implemented.

**No mocks. Ever.** `unittest.mock` is completely banned — no `patch`, no `MagicMock`, no `Mock`. Mocks make it trivially easy to write tests that pass but prove nothing. Instead, use fakes: real Python objects that implement the same abstract interface using in-memory data structures. Fakes behave like the real thing — they just store data in dicts and lists instead of databases.

**Listen to the tests.** If a test is hard to write, the test is telling you the design is wrong. Don't fight the test — fix the design. Difficulty in testing is the earliest signal of a coupling or abstraction problem.

**Tests speak domain language.** A domain expert should be able to read test names and understand what the system does. Test names are behavior specifications, not method inventories.

---

## The TDD Workflow

Every feature, bug fix, or behavior change follows this cycle. No exceptions.

### Step 1: Understand the Requirement

Before writing anything, be clear on what behavior you're implementing. If the requirement is ambiguous, ask the human. Don't guess.

Identify which layer owns the behavior:
- **Domain layer** — business rules, invariants, value object validation, aggregate state transitions
- **Service layer** — use case orchestration, coordinating domain objects through a UoW
- **Event publishing** — service function publishes events after commit for cross-module communication
- **Event handling** — handler in a subscribing module reacts to events from another module

### Step 2: Propose the Test

Before writing any test code, propose to the human:
- What behavior is being tested (in domain language)
- The test name
- The sample input and expected output
- Which layer the test targets

Wait for human approval. If you're unsure whether the test data is correct, say so and ask.

### Step 3: RED — Write a Failing Test

Write one test. Run it. It must fail. The failure must be for the right reason — a missing method, a wrong return value, an unraised exception. Not an import error, not a syntax error.

```bash
pytest tests/unit/domain/test_order.py::test_order_rejects_duplicate_line_items -x
```

Read the failure output. Confirm it fails because the behavior doesn't exist yet.

### Step 4: GREEN — Write Minimal Code

Write the simplest, most direct code that makes the test pass. Don't anticipate future requirements. Don't add error handling for cases that aren't tested. Don't refactor yet.

```bash
pytest tests/unit/domain/test_order.py::test_order_rejects_duplicate_line_items -x
```

The test passes. No other tests broke.

### Step 5: REFACTOR — Clean Up

Now improve the code. Remove duplication. Improve names. Extract helpers if the same logic appears in multiple places. But only if it actually appears — don't extract for hypothetical reuse.

```bash
pytest tests/unit/ -x
```

All tests still pass. The refactoring changed structure, not behavior.

### Step 6: Repeat

Go back to Step 2 with the next behavior. Each cycle adds one behavior to the system, proven by one test.

---

## What This Skill Covers

**In the TDD loop (run during feature development):**
- Domain unit tests — entities, value objects, aggregates, domain services
- Service layer tests — use case orchestration with fakes
- Event publishing tests — verify service functions publish the right events after commit (using FakeEventPublisher)
- Event handler tests — verify handlers in subscribing modules react correctly (using FakeUoW)

**Out of the TDD loop (run in dev/staging environment):**
- Integration tests — repository methods against a real database
- E2E tests — API endpoints through TestClient
- Stream integration tests — pub/sub round-trip through a real broker

The TDD workflow drives domain, service, and event tests. Integration, E2E, and stream tests are a separate concern, run separately, and are not part of the RED-GREEN-REFACTOR cycle during feature development.

---

## Test Naming

Tests are domain specifications. A domain expert reads them and understands system behavior.

**Happy paths** — behavior IS the expectation:
```python
def test_order_allocates_to_earliest_batch():
def test_payment_captures_authorized_amount():
def test_batch_quantity_decreases_after_allocation():
```

**Error/edge cases** — append expected outcome:
```python
def test_order_with_insufficient_stock_raises_out_of_stock():
def test_payment_with_expired_card_raises_payment_declined():
def test_allocation_with_unknown_sku_raises_invalid_sku():
```

The name must match what the test asserts. If the name says "raises out of stock", the test must assert `pytest.raises(OutOfStockError)`.

---

## Test Data

Use minimal, abstract data by default. Keep it simple and obvious:

```python
# Good — minimal, clear
batch = Batch("batch-1", "sku-1", qty=20, eta=None)
line = OrderLine("order-1", "sku-1", qty=2)

# Bad — unnecessarily realistic, distracting
batch = Batch("WH-2024-EAST-00147", "HERMAN-MILLER-AERON-CHAIR-BLK", qty=20, eta=datetime(2024, 3, 15))
```

Use domain-realistic data only when the human explicitly asks for it. If you're unsure whether sample data accurately represents a real scenario, ask the human before writing the test.

---

## Anti-Scam-Test Guards

These rules exist because AI agents can easily write tests that pass but prove nothing.

1. **Every test has a meaningful assertion.** No `assert True`. No `assert result is not None` unless nullability IS the behavior being tested.

2. **Assert on domain-relevant output.** Test what the system produces — a return value, a state change on an aggregate, a raised exception. Never assert "was this function called" or inspect internal implementation details.

3. **Test name matches the assertion.** If the test is named `test_order_allocates_to_earliest_batch`, the assertion must verify that the earliest batch was selected. Not just that "something was allocated".

4. **No testing implementation details.** Test the behavior, not the mechanism. Don't assert on internal data structures, private methods, or how many times something was called.

5. **`unittest.mock` is banned.** No `patch`, `MagicMock`, `Mock`, `PropertyMock`, `AsyncMock`, or any import from `unittest.mock`. Use fakes instead.

---

## Running Tests

Domain and service tests must run without any infrastructure — no database, no network, no environment variables:

```bash
# Run only domain + service tests (fast, no infra)
pytest tests/unit/ -x

# Run integration tests separately (needs real DB)
pytest tests/integration/ -x

# Run everything
pytest
```

---

## Topic Guides

- RED-GREEN-REFACTOR Workflow → resources/red-green-refactor.md
- Designing Fakes → resources/designing-fakes.md
- TDD for Event Streaming → resources/testing-events.md
- Anti-Patterns and Scam Test Prevention → resources/anti-patterns.md
- Worked Example: TDD for a DDD Feature → resources/worked-example.md
