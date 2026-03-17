# Anti-Patterns and Scam Test Prevention

These patterns produce tests that pass but prove nothing. They are especially common when AI agents write tests — the agent optimizes for "all tests green" rather than "all behavior verified". Every rule here exists to prevent that failure mode.

---

## The Scam Test

A scam test is a test that passes, looks legitimate, but doesn't actually verify the behavior it claims to test. It exists to satisfy a "write tests" requirement without doing any real work.

### How to spot a scam test

- The test name describes a behavior, but the assertion checks something unrelated
- The test uses `assert True` or `assert result is not None` as its only assertion
- The test mocks away the thing it's supposed to be testing
- The test asserts on mock call counts instead of outcomes
- The test creates objects but never calls the method under test
- The test would still pass if you deleted the production code

### The ultimate check

**Delete the production code the test is supposed to verify. Does the test still pass?** If yes, it's a scam. This is why RED is non-negotiable — watching the test fail before writing production code is the only way to prove the test actually tests what you think it does.

---

## Anti-Pattern 1: Testing That Nothing Happened

```python
# BAD — asserts nothing meaningful
def test_batch_exists():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    assert batch is not None

# BAD — asserts only that it doesn't crash
def test_allocate_runs():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)
    batch.allocate(line)  # no assertion
```

**Fix:** Assert on the observable outcome:

```python
# GOOD — verifies the actual effect
def test_batch_allocation_reduces_available_quantity():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)

    batch.allocate(line)

    assert batch.available_quantity == 15
```

---

## Anti-Pattern 2: Asserting on Internals

```python
# BAD — tests implementation detail, not behavior
def test_allocate_adds_to_internal_set():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)

    batch.allocate(line)

    assert line in batch._allocations  # reaching into private state
```

**Fix:** Assert on the public, domain-meaningful outcome:

```python
# GOOD — tests observable behavior
def test_batch_allocation_reduces_available_quantity():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)

    batch.allocate(line)

    assert batch.available_quantity == 15
```

If there's no public way to observe the effect, that's a design signal — the aggregate might need a query method.

---

## Anti-Pattern 3: Mock-Driven Testing

```python
# BAD — tests wiring, not behavior
from unittest.mock import MagicMock, patch

def test_allocate_calls_repository():
    mock_uow = MagicMock()
    mock_uow.batches.get_by_sku.return_value = [some_batch]

    await allocate("order-1", "sku-1", 10, mock_uow)

    mock_uow.batches.get_by_sku.assert_called_once_with("sku-1")
    mock_uow.commit.assert_called_once()
```

This test proves the service calls certain methods in a certain order. It does NOT prove the allocation actually works. If you change the service to use a different method name, the test breaks — even if the behavior is identical.

**Fix:** Use a fake. Test the outcome.

```python
# GOOD — tests that allocation actually happened
async def test_allocate_returns_batch_reference():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    result = await allocate("order-1", "sku-1", 10, uow)

    assert result.batchref == "batch-1"
    assert uow.committed is True
```

---

## Anti-Pattern 4: Test Name Lies About Content

```python
# BAD — name says "earliest batch" but assertion doesn't verify it
def test_allocates_to_earliest_batch():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    result = await allocate("order-1", "sku-1", 10, uow)

    assert result is not None  # doesn't check which batch!
```

**Fix:** The assertion must verify exactly what the name claims:

```python
# GOOD — name and assertion match
async def test_allocates_to_earliest_batch():
    uow = FakeUnitOfWork()
    today = date.today()
    tomorrow = today + timedelta(days=1)
    await create_batch("early", "sku-1", 100, today, uow)
    await create_batch("late", "sku-1", 100, tomorrow, uow)

    result = await allocate("order-1", "sku-1", 10, uow)

    assert result.batchref == "early"
```

---

## Anti-Pattern 5: Overly Broad Assertions

```python
# BAD — passes for any non-empty string
async def test_allocate_returns_reference():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    result = await allocate("order-1", "sku-1", 10, uow)

    assert isinstance(result.batchref, str)
```

**Fix:** Assert on the exact expected value:

```python
# GOOD — asserts on the specific expected outcome
async def test_allocate_returns_reference():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    result = await allocate("order-1", "sku-1", 10, uow)

    assert result.batchref == "batch-1"
```

---

## Anti-Pattern 6: Testing the Framework

```python
# BAD — tests that pytest.raises works, not that your code raises
def test_validation():
    with pytest.raises(Exception):
        OrderLine("order-1", "sku-1", qty=-1)
```

**Fix:** Assert on the specific domain exception:

```python
# GOOD — tests the specific exception your domain raises
def test_order_line_rejects_negative_quantity():
    with pytest.raises(ValueError, match="positive"):
        OrderLine("order-1", "sku-1", qty=-1)
```

Catching `Exception` proves nothing — any bug could raise an exception. Catching the specific exception type (and optionally matching the message) proves your domain validation works.

---

## Anti-Pattern 7: Tests Written After Implementation

Writing the test after the code means you never saw it fail. You don't know if:
- The test actually exercises the code path you think it does
- A bug in the test makes it pass accidentally
- The test would catch a regression if you changed the code

This is the most insidious anti-pattern because the test looks correct. The RED step exists specifically to prevent this — watching the test fail proves it's testing the right thing.

---

## Red Flags — Stop and Rethink

If any of these are true, something has gone wrong:

1. **You wrote production code before the test.** Start over — write the test first.
2. **The test passed on the first run.** You either have a redundant test or a broken test.
3. **You changed the test to make it pass.** Tests are specifications. If the test is wrong, discuss with the human — don't silently fix it.
4. **The test setup is longer than 10 lines.** Your design might be too coupled. Simplify the domain model or extract a builder/factory for test setup.
5. **You're testing more than one behavior in a test.** Split it into separate tests.
6. **You need to import `unittest.mock`.** Use a fake instead.
7. **The test reaches into private attributes** (`_internal_field`). Test through public methods.
8. **The test asserts on how many times a method was called.** That's mock thinking. Assert on outcomes.
