# TDD for Event Streaming

Event publishing and event handling follow the same RED-GREEN-REFACTOR cycle as everything else. The fakes are different — `FakeEventPublisher` instead of `FakeUnitOfWork` — but the workflow is identical: propose the test, watch it fail, make it pass, clean up.

---

## Two Things to TDD

1. **Publishing**: A service function in Module A publishes the right events after a successful commit
2. **Handling**: An event handler in Module B reacts correctly when it receives an event

Both are service-layer tests. Both run without infrastructure. Both use fakes.

---

## FakeEventPublisher

The `FakeEventPublisher` records every event that was published. Tests assert on the list.

```python
# infrastructure/streaming/publisher.py (lives alongside AbstractEventPublisher)
class FakeEventPublisher(AbstractEventPublisher):
    def __init__(self) -> None:
        self.published_events: list = []

    async def publish(self, event, subject: str | None = None) -> None:
        self.published_events.append(event)
```

It follows the same pattern as `FakeUnitOfWork` — a real object with real state, implementing the full abstract interface.

---

## TDD Cycle: Publishing Events

### The requirement

> "When an order line is allocated to a batch, other modules should be notified."

This means the `allocate` service function needs to publish an `Allocated` event after the UoW commits.

### Cycle 1: Allocate publishes an Allocated event

**Propose the test:**

> **Test name:** `test_allocate_publishes_allocated_event`
> **Behavior:** After successful allocation, an `Allocated` event is published with the correct data
> **Input:** Create a batch, allocate an order line
> **Expected output:** `publisher.published_events` contains one `Allocated` event with matching fields

Human approves.

**RED:**

```python
# tests/unit/service/test_allocation_events.py
from infrastructure.streaming.publisher import FakeEventPublisher
from shared.events.ordering_events import Allocated

async def test_allocate_publishes_allocated_event():
    uow = FakeUnitOfWork()
    publisher = FakeEventPublisher()
    await create_batch("batch-1", "sku-1", 100, None, uow)

    await allocate("order-1", "sku-1", 10, uow, publisher)

    assert len(publisher.published_events) == 1
    event = publisher.published_events[0]
    assert isinstance(event, Allocated)
    assert event.batchref == "batch-1"
    assert event.sku == "sku-1"
    assert event.quantity == 10
```

Run it:
```bash
pytest tests/unit/service/test_allocation_events.py::test_allocate_publishes_allocated_event -x
```

**Fails:** `TypeError: allocate() got an unexpected keyword argument 'publisher'` — the service function doesn't accept a publisher yet.

This is the right failure. The function signature needs to change.

**GREEN:**

```python
# service/ordering/allocation.py
async def allocate(
    order_id: str, sku: str, quantity: int,
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

    for event in uow.collect_new_events():
        await publisher.publish(event)

    return result
```

Run it. **Passes.**

Full suite: `pytest tests/unit/ -x` — **all green.**

**REFACTOR:** Nothing to clean up. Move to next behavior.

### Cycle 2: No events published on failure

**Propose the test:**

> **Test name:** `test_allocate_does_not_publish_events_on_failure`
> **Behavior:** When allocation fails (out of stock), no events are published
> **Input:** Create a batch with 1 unit, try to allocate 100
> **Expected output:** `publisher.published_events` is empty

**RED:**

```python
async def test_allocate_does_not_publish_events_on_failure():
    uow = FakeUnitOfWork()
    publisher = FakeEventPublisher()
    await create_batch("batch-1", "sku-1", 1, None, uow)

    with pytest.raises(OutOfStockError):
        await allocate("order-1", "sku-1", 100, uow, publisher)

    assert len(publisher.published_events) == 0
```

Run it. **Passes immediately** — the publishing code is after `async with uow:`, so if the UoW rolls back (domain exception), the loop never runs.

This test documents the contract explicitly: failures produce zero events. Still valuable as a specification.

---

## TDD Cycle: Event Handlers

Event handlers are service-layer functions in the subscribing module. They receive an event and their own UoW.

### The requirement

> "When fulfillment receives an Allocated event, it should create a pick list."

### Cycle 1: Handler creates a pick list

**Propose the test:**

> **Test name:** `test_handle_allocated_creates_pick_list`
> **Behavior:** Receiving an `Allocated` event creates a `PickList` in the fulfillment module
> **Input:** An `Allocated` event with order-1, sku-1, quantity 10
> **Expected output:** A `PickList` exists with matching data, UoW committed

Human approves.

**RED:**

```python
# tests/unit/handlers/test_fulfillment_handlers.py
from modules.fulfillment.handlers.external_events import handle_allocated
from shared.events.ordering_events import Allocated
from unit_of_work.fake import FakeUnitOfWork

async def test_handle_allocated_creates_pick_list():
    uow = FakeUnitOfWork()
    event = Allocated(order_id="order-1", batchref="batch-1", sku="sku-1", quantity=10)

    await handle_allocated(event, uow)

    assert uow.committed is True
    pick = await uow.pick_lists.get("order-1")
    assert pick is not None
    assert pick.sku == "sku-1"
    assert pick.quantity == 10
```

Run it:
```bash
pytest tests/unit/handlers/test_fulfillment_handlers.py::test_handle_allocated_creates_pick_list -x
```

**Fails:** `ImportError: cannot import name 'handle_allocated'` — the handler doesn't exist yet.

**GREEN:**

```python
# modules/fulfillment/handlers/external_events.py
from shared.events.ordering_events import Allocated
from unit_of_work.abstract import AbstractUnitOfWork
from modules.fulfillment.domain.model import PickList

async def handle_allocated(event: Allocated, uow: AbstractUnitOfWork) -> None:
    async with uow:
        pick = PickList(
            order_id=event.order_id,
            sku=event.sku,
            quantity=event.quantity,
        )
        await uow.pick_lists.add(pick)
```

Run it. **Passes.**

Full suite: **all green.**

### Cycle 2: Handler ignores duplicate events

**Propose the test:**

> **Test name:** `test_handle_allocated_ignores_duplicate_event`
> **Behavior:** If a pick list already exists for this order, the handler doesn't create a second one
> **Input:** Handle the same Allocated event twice
> **Expected output:** Only one pick list exists

This is an important edge case — event streams can deliver duplicates (at-least-once delivery). Handlers must be idempotent.

**RED:**

```python
async def test_handle_allocated_ignores_duplicate_event():
    uow = FakeUnitOfWork()
    event = Allocated(order_id="order-1", batchref="batch-1", sku="sku-1", quantity=10)

    await handle_allocated(event, uow)
    await handle_allocated(event, uow)

    picks = await uow.pick_lists.list_for_order("order-1")
    assert len(picks) == 1
```

**Fails:** The handler creates a duplicate.

**GREEN:** Add an idempotency check to the handler:

```python
async def handle_allocated(event: Allocated, uow: AbstractUnitOfWork) -> None:
    async with uow:
        existing = await uow.pick_lists.get(event.order_id)
        if existing:
            return
        pick = PickList(order_id=event.order_id, sku=event.sku, quantity=event.quantity)
        await uow.pick_lists.add(pick)
```

**Passes.** Full suite green.

---

## What to Assert On

### For event publishing tests

| Good assertion | Bad assertion |
|---|---|
| `assert isinstance(event, Allocated)` | `assert publisher.publish.called` (mock thinking) |
| `assert event.sku == "sku-1"` | `assert len(publisher.published_events) > 0` (too weak) |
| `assert event.quantity == 10` | `assert isinstance(event, object)` (meaningless) |
| `assert len(publisher.published_events) == 1` | No assertion at all |

### For event handler tests

| Good assertion | Bad assertion |
|---|---|
| `assert pick.sku == "sku-1"` | `assert uow.pick_lists.add.called` (mock thinking) |
| `assert uow.committed is True` | `assert True` |
| `assert len(picks) == 1` (idempotency) | Testing that the handler called specific internal methods |

---

## The Human Checkpoint for Events

Before writing any event-related test, propose to the human:

1. **What event should this operation publish?** Name it, list the fields.
2. **Which modules should react?** What should each handler do?
3. **What about duplicates?** Should the handler be idempotent? (Almost always yes.)
4. **What about ordering?** Does it matter if events arrive out of order?

These are design decisions that affect the test. Get human approval before writing.

---

## Anti-Patterns Specific to Event Testing

### Testing the broker instead of the behavior

```python
# BAD — tests that publish was called (mock thinking applied to fakes)
async def test_allocate_calls_publish():
    ...
    assert hasattr(publisher, 'published_events')  # proves nothing
```

**Fix:** Assert on the content of the published event, not that publishing happened.

### Testing serialization in unit tests

```python
# BAD — serialization is infrastructure, not behavior
async def test_event_serializes_to_json():
    event = Allocated(order_id="o1", batchref="b1", sku="s1", quantity=10)
    data = serialize_event(event)
    assert b'"event_type"' in data
```

Serialization correctness is an integration test concern. In the TDD loop, test that the right event object is published. Whether it serializes correctly over the wire is a separate test that runs in the dev environment.

### Forgetting idempotency

Event streams guarantee at-least-once delivery. Handlers will receive duplicates. If your handler test doesn't cover the "handle same event twice" case, the handler will create duplicate data in production.

Always include an idempotency test for every event handler.
