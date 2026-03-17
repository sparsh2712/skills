# RED-GREEN-REFACTOR Workflow

The cycle is mechanical. Follow it exactly. Each pass adds one behavior to the system.

---

## RED — Write a Failing Test

You are defining what the system should do before it does it. The test is a specification.

### What to write

One test. One behavior. The test calls the code the way a real caller would — through the public API of a domain object or through a service function with a fake UoW.

```python
# tests/unit/domain/test_batch.py
def test_batch_allocation_reduces_available_quantity():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)

    batch.allocate(line)

    assert batch.available_quantity == 15
```

### What to verify

Run the test. It must fail.

```bash
pytest tests/unit/domain/test_batch.py::test_batch_allocation_reduces_available_quantity -x
```

Read the failure output carefully. The failure should be because:
- The method doesn't exist yet (`AttributeError`)
- The method exists but returns the wrong value (`AssertionError`)
- The expected exception isn't raised

The failure should NOT be because:
- Import errors (your test file is broken, not the code)
- Syntax errors
- Wrong test setup

If the test fails for the wrong reason, fix the test first. Don't proceed to GREEN with a broken test.

### If the test passes immediately

Something is wrong. Either:
- The behavior already exists (you're writing a redundant test)
- Your test isn't actually testing what you think it is
- Your assertion is too weak

Investigate before moving on. A test that never failed proves nothing.

---

## GREEN — Make It Pass

Write the absolute minimum code to make the test pass. This is not the time for elegance.

### Rules for GREEN

**Do the simplest thing.** If the test expects `available_quantity == 15`, you could hardcode `return 15` as a first pass. That's fine — the next test will force you to generalize.

**Don't add untested behavior.** If your test doesn't check error handling, don't add error handling. If your test doesn't check edge cases, don't handle edge cases. Those are future tests.

**Don't refactor yet.** Ugly code that passes is better than beautiful code you haven't proven works. The refactoring step exists specifically for cleanup.

```python
# domain/ordering/model.py
class Batch:
    def __init__(self, reference: str, sku: str, qty: int, eta: date | None):
        self.reference = reference
        self.sku = sku
        self._purchased_quantity = qty
        self._allocations: set[OrderLine] = set()

    def allocate(self, line: OrderLine) -> None:
        self._allocations.add(line)

    @property
    def available_quantity(self) -> int:
        return self._purchased_quantity - sum(l.qty for l in self._allocations)
```

### What to verify

Run the test again:

```bash
pytest tests/unit/domain/test_batch.py::test_batch_allocation_reduces_available_quantity -x
```

It passes. Now run the full test suite to make sure you didn't break anything:

```bash
pytest tests/unit/ -x
```

All green. Move to REFACTOR.

---

## REFACTOR — Clean Up

Now you have working, tested code. Improve its structure without changing its behavior.

### What to look for

- **Duplication** — same logic in two places? Extract it.
- **Bad names** — does the variable name describe what it holds? Does the method name describe what it does in domain language?
- **Long methods** — can you extract a named step that makes the code read like a sentence?
- **Test cleanup** — is the test readable? Can you extract a helper to reduce setup noise?

### What NOT to do

- Don't add new behavior. That's a new RED step.
- Don't add error handling that isn't tested. That's a new RED step.
- Don't extract abstractions for hypothetical future reuse.
- Don't rename things to be "more general" — keep domain-specific names.

### What to verify

Run the full unit test suite:

```bash
pytest tests/unit/ -x
```

All tests still pass. Your refactoring preserved behavior.

---

## Domain Layer TDD

Domain objects are pure Python. Tests are the simplest kind — instantiate, call, assert.

### Testing entities and aggregates

```python
def test_batch_cannot_allocate_more_than_available():
    batch = Batch("batch-1", "sku-1", qty=5, eta=None)
    line = OrderLine("order-1", "sku-1", qty=10)

    with pytest.raises(OutOfStockError):
        batch.allocate(line)

def test_batch_allocation_is_idempotent():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)

    batch.allocate(line)
    batch.allocate(line)

    assert batch.available_quantity == 15
```

### Testing value objects

Value objects validate at construction and support equality:

```python
def test_order_line_requires_positive_quantity():
    with pytest.raises(ValueError):
        OrderLine("order-1", "sku-1", qty=0)

def test_order_lines_with_same_values_are_equal():
    line_a = OrderLine("order-1", "sku-1", qty=5)
    line_b = OrderLine("order-1", "sku-1", qty=5)

    assert line_a == line_b

def test_order_lines_with_different_quantities_are_not_equal():
    line_a = OrderLine("order-1", "sku-1", qty=5)
    line_b = OrderLine("order-1", "sku-1", qty=10)

    assert line_a != line_b
```

### Testing domain services

Domain services are stateless functions that operate on domain objects:

```python
def test_allocate_prefers_earlier_batch():
    today = date.today()
    tomorrow = today + timedelta(days=1)
    early = Batch("early", "sku-1", qty=10, eta=today)
    late = Batch("late", "sku-1", qty=10, eta=tomorrow)
    line = OrderLine("order-1", "sku-1", qty=5)

    ref = allocate_to_earliest_batch([late, early], line)

    assert ref == "early"
```

---

## Service Layer TDD

Service tests verify use case orchestration through fakes. The service function receives a `FakeUnitOfWork` and operates exactly as it would with a real one.

### The pattern

```python
# tests/unit/service/test_allocation.py
async def test_allocate_returns_batch_reference():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    result = await allocate("order-1", "sku-1", 10, uow)

    assert result.batchref == "batch-1"

async def test_allocate_commits_on_success():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    await allocate("order-1", "sku-1", 10, uow)

    assert uow.committed is True

async def test_allocate_with_invalid_sku_raises_and_does_not_commit():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    with pytest.raises(InvalidSkuError):
        await allocate("order-1", "nonexistent-sku", 10, uow)

    assert uow.committed is False
```

### What to test at the service level

- The happy path returns the expected result
- `uow.committed is True` after successful operations
- `uow.committed is False` when domain exceptions are raised
- Domain exceptions propagate through the service correctly
- The service loads the right data from repositories and passes it to domain objects

### What NOT to test at the service level

- Business rules (those belong in domain tests)
- SQL queries (those belong in integration tests)
- HTTP status codes (those belong in E2E tests)
