# Worked Example: TDD for a DDD Feature

This walkthrough shows the complete TDD cycle for adding a new feature to a DDD backend. Each step follows the RED-GREEN-REFACTOR workflow exactly.

---

## The Feature

**Requirement:** Add a `deallocate` capability to the stock allocation system. A customer cancels an order line, and the allocated quantity should be returned to the batch.

**Layer analysis:**
- Domain behavior: `Batch.deallocate(line)` — returns quantity to available stock
- Service orchestration: `deallocate(order_id, sku, qty, uow)` — finds the right batch, calls domain method, commits

---

## Cycle 1: Domain — Batch Can Deallocate

### Propose the test (Step 2)

> **Test name:** `test_batch_deallocation_increases_available_quantity`
> **Behavior:** Deallocating a previously allocated line returns that quantity to the batch
> **Input:** Batch with 20 units, allocate 5, then deallocate those 5
> **Expected output:** `available_quantity` returns to 20

Human approves.

### RED (Step 3)

```python
# tests/unit/domain/test_batch.py
def test_batch_deallocation_increases_available_quantity():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)
    batch.allocate(line)

    batch.deallocate(line)

    assert batch.available_quantity == 20
```

Run it:
```bash
pytest tests/unit/domain/test_batch.py::test_batch_deallocation_increases_available_quantity -x
```

**Fails:** `AttributeError: 'Batch' object has no attribute 'deallocate'`

This is the right failure — the method doesn't exist yet.

### GREEN (Step 4)

```python
# domain/ordering/model.py
class Batch:
    # ... existing code ...

    def deallocate(self, line: OrderLine) -> None:
        self._allocations.discard(line)
```

Run it:
```bash
pytest tests/unit/domain/test_batch.py::test_batch_deallocation_increases_available_quantity -x
```

**Passes.**

Run the full suite:
```bash
pytest tests/unit/ -x
```

**All green.**

### REFACTOR (Step 5)

The code is simple enough. Nothing to refactor. Move to the next behavior.

---

## Cycle 2: Domain — Deallocating Unallocated Line Does Nothing

### Propose the test

> **Test name:** `test_batch_deallocate_unallocated_line_is_noop`
> **Behavior:** Deallocating a line that was never allocated doesn't change anything
> **Input:** Batch with 20 units, deallocate a line that was never allocated
> **Expected output:** `available_quantity` remains 20

Human approves.

### RED

```python
def test_batch_deallocate_unallocated_line_is_noop():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    line = OrderLine("order-1", "sku-1", qty=5)

    batch.deallocate(line)

    assert batch.available_quantity == 20
```

Run it:
```bash
pytest tests/unit/domain/test_batch.py::test_batch_deallocate_unallocated_line_is_noop -x
```

**Passes immediately.**

This is fine — `set.discard()` already handles missing elements silently. The test is still valuable as a specification: it documents that deallocating an unallocated line is intentionally a no-op, not a bug.

---

## Cycle 3: Service — Deallocate Use Case

### Propose the test

> **Test name:** `test_deallocate_returns_batch_reference`
> **Behavior:** The deallocate service finds the batch that holds the allocation and returns its reference
> **Input:** Create batch "batch-1" with sku-1, allocate order-1 for 5 units, then deallocate order-1
> **Expected output:** Returns "batch-1"

Human approves.

### RED

```python
# tests/unit/service/test_allocation.py
async def test_deallocate_returns_batch_reference():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)
    await allocate("order-1", "sku-1", 5, uow)

    ref = await deallocate("order-1", "sku-1", 5, uow)

    assert ref == "batch-1"
```

Run it:
```bash
pytest tests/unit/service/test_allocation.py::test_deallocate_returns_batch_reference -x
```

**Fails:** `ImportError: cannot import name 'deallocate' from 'service.ordering.allocation'`

Right failure — the service function doesn't exist yet.

### GREEN

```python
# service/ordering/allocation.py
async def deallocate(order_id: str, sku: str, quantity: int, uow: AbstractUnitOfWork) -> str:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        for batch in batches:
            if line in batch._allocations:
                batch.deallocate(line)
                return batch.reference
        raise BatchNotFoundError(f"No batch found with allocation for {order_id}/{sku}")
```

Run it:
```bash
pytest tests/unit/service/test_allocation.py::test_deallocate_returns_batch_reference -x
```

**Passes.**

Full suite:
```bash
pytest tests/unit/ -x
```

**All green.**

### REFACTOR

Looking at the GREEN code — it reaches into `batch._allocations` to check membership. That's an internal detail. The batch should expose this as a domain method.

But that's new behavior, so it gets its own RED step. For now, the code works and the test passes. Note the improvement for the next cycle.

---

## Cycle 4: Domain — Batch Knows If Line Is Allocated

### Propose the test

> **Test name:** `test_batch_reports_whether_line_is_allocated`
> **Behavior:** Batch can tell you if a specific line is currently allocated to it
> **Input:** Allocate a line to a batch, ask if it's allocated
> **Expected output:** True for allocated line, False for unallocated line

Human approves.

### RED

```python
def test_batch_reports_whether_line_is_allocated():
    batch = Batch("batch-1", "sku-1", qty=20, eta=None)
    allocated_line = OrderLine("order-1", "sku-1", qty=5)
    other_line = OrderLine("order-2", "sku-1", qty=3)
    batch.allocate(allocated_line)

    assert batch.has_allocation(allocated_line) is True
    assert batch.has_allocation(other_line) is False
```

**Fails:** `AttributeError: 'Batch' object has no attribute 'has_allocation'`

### GREEN

```python
# domain/ordering/model.py
class Batch:
    def has_allocation(self, line: OrderLine) -> bool:
        return line in self._allocations
```

**Passes.**

### REFACTOR

Now go back and clean up the service function from Cycle 3:

```python
# service/ordering/allocation.py — refactored
async def deallocate(order_id: str, sku: str, quantity: int, uow: AbstractUnitOfWork) -> str:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        for batch in batches:
            if batch.has_allocation(line):
                batch.deallocate(line)
                return batch.reference
        raise BatchNotFoundError(f"No batch found with allocation for {order_id}/{sku}")
```

Run full suite:
```bash
pytest tests/unit/ -x
```

**All green.** The refactoring replaced `line in batch._allocations` (internal detail) with `batch.has_allocation(line)` (public domain method). Same behavior, better design.

---

## Cycle 5: Service — Deallocate Commits

### Propose the test

> **Test name:** `test_deallocate_commits_on_success`
> **Behavior:** Successful deallocation commits the unit of work
> **Input:** Same setup as Cycle 3
> **Expected output:** `uow.committed is True`

### RED

```python
async def test_deallocate_commits_on_success():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)
    await allocate("order-1", "sku-1", 5, uow)
    uow.committed = False  # reset after allocate's commit

    await deallocate("order-1", "sku-1", 5, uow)

    assert uow.committed is True
```

**Passes immediately** — the `async with uow:` context manager already commits on clean exit.

This is acceptable. The test documents the expectation explicitly.

---

## Cycle 6: Service — Deallocate Raises When Not Found

### Propose the test

> **Test name:** `test_deallocate_with_no_matching_allocation_raises_batch_not_found`
> **Behavior:** Trying to deallocate a line that isn't allocated anywhere raises BatchNotFoundError
> **Input:** Create a batch but don't allocate anything, try to deallocate
> **Expected output:** `BatchNotFoundError` raised

### RED

```python
async def test_deallocate_with_no_matching_allocation_raises_batch_not_found():
    uow = FakeUnitOfWork()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    with pytest.raises(BatchNotFoundError):
        await deallocate("order-1", "sku-1", 5, uow)
```

**Passes immediately** — already handled in the GREEN code from Cycle 3.

---

## Summary of What Was Built

In 6 TDD cycles:
- 4 domain behaviors: deallocate, noop on unallocated, has_allocation query, value object equality
- 2 service behaviors: deallocate orchestration, error handling
- 1 refactoring: replaced internal set access with public domain method
- Zero mocks used
- All tests run instantly with no infrastructure

The tests now serve as living documentation. Anyone reading them can understand exactly what `deallocate` does, what happens in edge cases, and what errors are possible — all in domain language.
