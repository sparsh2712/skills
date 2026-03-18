# Testing — DDD Test Strategy

---

## The Testing Pyramid for DDD

Most tests live at the domain and service layers. They are fast, database-free, and test real business rules. Integration tests are few and focused on proving the infrastructure wiring works.

```
        ╱╲
       ╱  ╲       Integration tests (few)
      ╱    ╲      Real database, real UoW, real repositories
     ╱──────╲
    ╱        ╲    Service layer tests (moderate)
   ╱          ╲   FakeUoW, test use case orchestration
  ╱────────────╲
 ╱              ╲  Domain unit tests (most)
╱                ╲ Pure Python, no fakes needed, instant
╱──────────────────╲
```

---

## Domain Unit Tests

Domain objects are plain Python — test them directly with no setup. These tests are the fastest and most valuable. They prove that business invariants hold.

```python
# tests/unit/domain/test_batch.py
import pytest
from datetime import datetime, timedelta
from domain.ordering.model import Batch, OrderLine
from domain.ordering.exceptions import OutOfStockError

def test_can_allocate_if_available_greater_than_required():
    batch = Batch("batch-001", "SMALL-TABLE", 20, eta=None)
    line = OrderLine("order-001", "SMALL-TABLE", 2)

    batch.allocate(line)

    assert batch.available_quantity == 18

def test_cannot_allocate_if_available_less_than_required():
    batch = Batch("batch-001", "SMALL-TABLE", 1, eta=None)
    line = OrderLine("order-001", "SMALL-TABLE", 2)

    with pytest.raises(OutOfStockError):
        batch.allocate(line)

def test_can_deallocate():
    batch = Batch("batch-001", "SMALL-TABLE", 20, eta=None)
    line = OrderLine("order-001", "SMALL-TABLE", 2)
    batch.allocate(line)

    batch.deallocate(line)

    assert batch.available_quantity == 20

def test_allocation_is_idempotent():
    batch = Batch("batch-001", "SMALL-TABLE", 20, eta=None)
    line = OrderLine("order-001", "SMALL-TABLE", 2)
    batch.allocate(line)
    batch.allocate(line)

    assert batch.available_quantity == 18

def test_prefers_earlier_batches():
    today = datetime.now()
    later = today + timedelta(days=10)
    earliest = Batch("speedy", "CHAIR", 100, eta=today)
    latest = Batch("slow", "CHAIR", 100, eta=later)

    assert not (earliest > latest)
    assert latest > earliest
```

**What to test at this level:**
- Every invariant the aggregate enforces
- Value object validation (construction with invalid data raises)
- Equality and identity semantics
- State transitions and their guard conditions

---

## Value Object Tests

Value objects validate at construction. Test the happy path and every rejection.

```python
# tests/unit/domain/test_value_objects.py
import pytest
from domain.ordering.model import OrderLine

def test_orderline_requires_positive_quantity():
    with pytest.raises(ValueError, match="positive"):
        OrderLine("order-001", "CHAIR", 0)

    with pytest.raises(ValueError, match="positive"):
        OrderLine("order-001", "CHAIR", -5)

def test_orderlines_with_same_values_are_equal():
    line1 = OrderLine("order-001", "CHAIR", 10)
    line2 = OrderLine("order-001", "CHAIR", 10)

    assert line1 == line2

def test_orderlines_with_different_values_are_not_equal():
    line1 = OrderLine("order-001", "CHAIR", 10)
    line2 = OrderLine("order-001", "CHAIR", 5)

    assert line1 != line2
```

---

## Service Layer Tests with FakeUoW

Service tests prove that use cases correctly orchestrate domain objects and commit via UoW. They use `FakeUnitOfWork` — no database, instant execution.

```python
# tests/unit/service/test_allocation.py
import pytest
from datetime import datetime, timedelta
from service.ordering.allocation import allocate, create_batch, deallocate
from unit_of_work.fake import FakeUnitOfWork
from domain.ordering.exceptions import InvalidSkuError, OutOfStockError, BatchNotFoundError

async def test_allocates_to_earliest_available_batch():
    uow = FakeUnitOfWork()
    today = datetime.now()
    later = today + timedelta(days=10)
    await create_batch("speedy", "SMALL-TABLE", 100, today, uow)
    await create_batch("slow", "SMALL-TABLE", 100, later, uow)

    result = await allocate("order-001", "SMALL-TABLE", 10, uow)

    assert result.batchref == "speedy"

async def test_raises_for_invalid_sku():
    uow = FakeUnitOfWork()
    await create_batch("b1", "REAL-SKU", 100, None, uow)

    with pytest.raises(InvalidSkuError):
        await allocate("o1", "NONEXISTENT", 10, uow)

async def test_commits_on_successful_allocation():
    uow = FakeUnitOfWork()
    await create_batch("b1", "CHAIR", 100, None, uow)

    await allocate("o1", "CHAIR", 10, uow)

    assert uow.committed is True

async def test_does_not_commit_on_error():
    uow = FakeUnitOfWork()
    await create_batch("b1", "CHAIR", 1, None, uow)

    with pytest.raises(OutOfStockError):
        await allocate("o1", "CHAIR", 100, uow)

    assert uow.committed is False

async def test_deallocate_returns_batch_reference():
    uow = FakeUnitOfWork()
    await create_batch("b1", "DESK", 100, None, uow)
    await allocate("o1", "DESK", 10, uow)

    ref = await deallocate("o1", "DESK", 10, uow)

    assert ref == "b1"

async def test_deallocate_raises_when_not_found():
    uow = FakeUnitOfWork()
    await create_batch("b1", "DESK", 100, None, uow)

    with pytest.raises(BatchNotFoundError):
        await deallocate("o1", "DESK", 10, uow)
```

**What to test at this level:**
- Use case happy paths return expected results
- `uow.committed is True` after successful operations
- `uow.committed is False` when domain exceptions are raised
- Each domain exception surfaces through the service layer correctly

---

## Event Publishing Tests

When a service function publishes events after commit, use `FakeEventPublisher` to verify the right events were published:

```python
# tests/unit/service/test_allocation_events.py
import pytest
from service.ordering.allocation import allocate, create_batch
from unit_of_work.fake import FakeUnitOfWork
from infrastructure.streaming.publisher import FakeEventPublisher
from shared.events.ordering_events import Allocated

async def test_allocate_publishes_allocated_event():
    uow = FakeUnitOfWork()
    publisher = FakeEventPublisher()
    await create_batch("b1", "CHAIR", 100, None, uow)

    await allocate("o1", "CHAIR", 10, uow, publisher)

    assert len(publisher.published_events) == 1
    event = publisher.published_events[0]
    assert isinstance(event, Allocated)
    assert event.batchref == "b1"
    assert event.sku == "CHAIR"
    assert event.quantity == 10

async def test_no_events_published_on_failure():
    uow = FakeUnitOfWork()
    publisher = FakeEventPublisher()
    await create_batch("b1", "CHAIR", 1, None, uow)

    with pytest.raises(Exception):
        await allocate("o1", "CHAIR", 100, uow, publisher)

    assert len(publisher.published_events) == 0
```

**What to test:**
- The correct event type is published after a successful operation
- Event fields contain the expected data
- No events are published when the operation fails

---

## Event Handler Tests

Event handlers in subscribing modules are tested the same way as service functions — with FakeUoW:

```python
# tests/unit/handlers/test_fulfillment_handlers.py
from modules.fulfillment.handlers.external_events import handle_allocated
from shared.events.ordering_events import Allocated
from unit_of_work.fake import FakeUnitOfWork

async def test_handle_allocated_creates_pick_list():
    uow = FakeUnitOfWork()
    event = Allocated(order_id="o1", batchref="b1", sku="CHAIR", quantity=10)

    await handle_allocated(event, uow)

    assert uow.committed is True
    pick = await uow.pick_lists.get("o1")
    assert pick is not None
    assert pick.sku == "CHAIR"
```

**What to test:**
- Handler creates the right domain objects from the event data
- Handler commits on success
- Handler raises the right exception for invalid/duplicate events

---

## Integration Tests

Integration tests verify that the real database, real repositories, and real UoW work together. They are slow and few. Use them to prove infrastructure wiring, not business logic.

```python
# tests/integration/test_repository.py
import pytest
from sqlalchemy.ext.asyncio import AsyncEngine
from unit_of_work.concrete import UnitOfWork
from domain.ordering.model import Batch, OrderLine

@pytest.fixture
async def engine(test_database) -> AsyncEngine:
    """Provided by your test infrastructure — creates a real async engine."""
    return test_database.engine

async def test_can_save_and_retrieve_batch(engine: AsyncEngine):
    uow = UnitOfWork(engine)
    async with uow:
        await uow.batches.add(Batch("batch-001", "CHAIR", 100, eta=None))

    async with uow:
        batch = await uow.batches.get("batch-001")
        assert batch is not None
        assert batch.sku == "CHAIR"
        assert batch.available_quantity == 100

async def test_repository_tracks_allocations(engine: AsyncEngine):
    uow = UnitOfWork(engine)
    async with uow:
        batch = Batch("batch-001", "CHAIR", 100, eta=None)
        batch.allocate(OrderLine("order-001", "CHAIR", 10))
        await uow.batches.add(batch)

    async with uow:
        batch = await uow.batches.get("batch-001")
        assert batch.available_quantity == 90
```

**What to test at this level:**
- Round-trip persistence: save an aggregate, reload it, verify state
- Query methods return correct results from real SQL
- Transaction rollback works on failure
- Repository `_to_domain` mapping is correct

---

## Test Organization

```
tests/
  unit/
    domain/
      test_batch.py              # Aggregate invariants, value objects, events
      test_order.py               # If you have an Order aggregate
    service/
      test_allocation.py          # Use case tests with FakeUoW
      test_allocation_events.py   # Event publishing tests with FakeEventPublisher
    handlers/
      test_fulfillment_handlers.py  # Event handler tests
  integration/
    test_repository.py            # Real DB round-trip tests
    conftest.py                   # Database fixtures, test engine setup
  conftest.py                     # Shared fixtures, pytest-asyncio config
```

---

## pytest Configuration

```ini
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

`asyncio_mode = "auto"` means you do not need `@pytest.mark.asyncio` on every async test function — pytest-asyncio handles it automatically.

---

## Rules

- **Test domain invariants at the domain level** — not through service functions. If `Batch.allocate` has a business rule, test it directly on `Batch`.
- **Test orchestration at the service level** — the service test proves that `allocate()` loads the right batches, calls the right domain method, and commits.
- **Never test business logic via HTTP** — router tests should only verify request/response mapping and status codes, not domain rules.
- **One assert per concept** — a test can have multiple asserts if they verify the same logical outcome. Do not cram unrelated checks into one test.
- **Name tests after the behavior, not the method** — `test_cannot_allocate_more_than_available` not `test_allocate_raises`.
