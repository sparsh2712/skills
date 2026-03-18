# Service Layer

---

## Purpose

The service layer defines **where each use case begins and ends**. It sits between the infrastructure layer and the interaction layer.

A service function:
- Accepts primitives and a UoW — not domain objects, not HTTP requests
- Loads aggregates via `uow.{repo}`
- Delegates business rules to the domain model
- Returns primitives or simple dataclasses — not domain objects
- Is callable from HTTP router, CLI, or task queue without modification

---

## Service Function Structure

```python
# service/ordering/allocation.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
from unit_of_work.abstract import AbstractUnitOfWork
from domain.ordering.model import Batch, OrderLine
from domain.ordering.services import allocate_to_earliest_batch
from domain.ordering.exceptions import InvalidSkuError

@dataclass
class AllocationResult:
    batchref: str
    sku: str

async def create_batch(
    reference: str,
    sku: str,
    quantity: int,
    eta: Optional[datetime],
    uow: AbstractUnitOfWork,
) -> str:
    async with uow:
        await uow.batches.add(Batch(reference=reference, sku=sku, purchased_quantity=quantity, eta=eta))
        return reference

async def allocate(
    order_id: str,
    sku: str,
    quantity: int,
    uow: AbstractUnitOfWork,
) -> AllocationResult:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        if not batches:
            raise InvalidSkuError(f"Invalid sku {sku}")
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        batchref = allocate_to_earliest_batch(batches, line)
        return AllocationResult(batchref=batchref, sku=sku)

async def deallocate(
    order_id: str,
    sku: str,
    quantity: int,
    uow: AbstractUnitOfWork,
) -> str:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        batch = next((b for b in batches if line in b._allocations), None)
        if batch is None:
            raise AllocationNotFoundError(f"No allocation for order {order_id} sku {sku}")
        batch.deallocate(line)
        await uow.batches.save(batch)
        return batch.reference
```

---

## Primitives In, Primitives Out

Service functions accept **primitives** (`str`, `int`, `Decimal`, `date`), not domain objects. This keeps callers decoupled from domain internals. The service constructs domain objects internally. The caller never touches domain classes.

```python
# Good — primitives in, simple dataclass out
async def allocate(order_id: str, sku: str, quantity: int, uow: AbstractUnitOfWork) -> AllocationResult: ...

# Bad — domain object in (couples callers to domain internals)
async def allocate(line: OrderLine, uow: AbstractUnitOfWork) -> AllocationResult: ...
```

---

## Application Service vs Domain Service

**Application Service** (this layer): Thin orchestrator. Loads aggregates, calls domain, commits via UoW, returns primitives. Has no business logic of its own.

**Domain Service** (`domain/{context}/services.py`): Stateless computation that belongs in the domain but doesn't fit on any Entity or Aggregate. No database access. Pure Python.

```python
# domain/ordering/services.py — pure domain logic, no infra
from domain.ordering.model import Batch, OrderLine
from domain.ordering.exceptions import OutOfStockError

def allocate_to_earliest_batch(batches: list[Batch], line: OrderLine) -> str:
    try:
        batch = next(b for b in sorted(batches) if b.can_allocate(line))
    except StopIteration:
        raise OutOfStockError(f"Out of stock for sku {line.sku}")
    batch.allocate(line)
    return batch.reference
```

---

## Unit Testing the Service Layer

With `FakeUnitOfWork`, service tests are fast and database-free:

```python
# tests/unit/service/test_allocation.py
import pytest
from datetime import datetime, timedelta
from service.ordering.allocation import allocate, create_batch
from unit_of_work.fake import FakeUnitOfWork
from domain.ordering.exceptions import InvalidSkuError, OutOfStockError

async def test_allocates_to_earliest_batch():
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
    await create_batch("b1", "POPULAR-CHAIR", 100, None, uow)

    await allocate("o1", "POPULAR-CHAIR", 10, uow)

    assert uow.committed is True
```

---

## What Belongs in the Service Layer

| Belongs | Does NOT belong |
|---|---|
| Loading aggregates via UoW | Business rule validation |
| Calling domain services | Direct SQL queries |
| Constructing domain objects from primitives | `conn.commit()` |
| Returning simple dataclasses or primitives | HTTP status codes |
| Raising domain exceptions | Web framework imports |
| Calling `uow.{repo}.save()` | Domain logic |
| Publishing events after commit | Subscribing to events (that's module registration) |

---

## Publishing Events After Commit

When a module needs to notify other modules about what happened, the service function publishes events after the UoW commits successfully:

```python
# service/ordering/allocation.py
from infrastructure.streaming.publisher import AbstractEventPublisher

async def allocate(
    order_id: str,
    sku: str,
    quantity: int,
    uow: AbstractUnitOfWork,
    publisher: AbstractEventPublisher,
) -> AllocationResult:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        if not batches:
            raise InvalidSkuError(f"Invalid sku {sku}")
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        batchref = allocate_to_earliest_batch(batches, line)
        result = AllocationResult(batchref=batchref, sku=sku)

    # UoW committed — publish events for other modules
    for event in uow.collect_new_events():
        await publisher.publish(event)

    return result
```

The publisher is injected the same way as UoW — through dependency injection. The service function never knows what broker is behind it. In tests, use `FakeEventPublisher` and assert on `publisher.published_events`.

If the UoW rolls back (domain exception), the code after `async with uow:` never runs. No events are published for failed operations.

---

## Event Handlers (Cross-Module)

Event handlers react to events from other modules. They live in `modules/{name}/handlers/` and receive an event plus their own UoW — an independent transaction.

```python
# modules/fulfillment/handlers/external_events.py
from shared.events.ordering_events import Allocated
from unit_of_work.abstract import AbstractUnitOfWork

async def handle_allocated(event: Allocated, uow: AbstractUnitOfWork) -> None:
    async with uow:
        pick = PickList(
            order_id=event.order_id,
            sku=event.sku,
            quantity=event.quantity,
        )
        await uow.pick_lists.add(pick)
```

Event handlers follow the same rules as service functions: accept primitives (the event dataclass) and a UoW, delegate to domain objects, let the UoW commit. See resources/event-streaming.md for the full pattern and resources/module-structure.md for how handlers are registered.
