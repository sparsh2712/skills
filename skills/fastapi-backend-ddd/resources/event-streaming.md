# Event Streaming — Cross-Module Communication

---

## Purpose

Event streaming is how modules talk to each other without importing each other's code. When something happens in one module (an order is placed, stock is allocated), it publishes an event to a persistent stream. Other modules subscribe to events they care about and react independently.

Within a module, service functions are called directly — no events needed. Events exist only for cross-module communication.

---

## The Flow

```
Module A (ordering):
  Router → service function → domain logic → UoW commits
    → events collected from aggregates
    → events serialized and published to stream
    → done — Module A doesn't know or care who's listening

Module B (fulfillment):
  Subscriber receives event from stream
    → deserializes into event dataclass
    → calls event handler function
    → handler uses its own UoW (independent transaction)
    → done
```

The key rule: **events are published only after a successful commit.** If the UoW rolls back, no events are published. This prevents other modules from reacting to something that didn't actually happen.

---

## Event Collection from Aggregates

Domain aggregates append events to `self.events` during state changes (this is already defined in resources/domain-model.md). After the UoW commits, these events need to be collected and published.

The `collect_new_events()` method on AbstractUnitOfWork drains events from every aggregate the repositories have seen:

```python
# Added to AbstractUnitOfWork (see resources/unit-of-work.md)
def collect_new_events(self):
    for repo in self._repositories():
        for aggregate in repo.seen:
            while aggregate.events:
                yield aggregate.events.pop(0)
```

This is the bridge between "domain raises events" and "events get published to the stream."

---

## Publishing After Commit

The service function publishes events after the UoW context manager exits cleanly:

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

    # UoW committed successfully — now publish events
    for event in uow.collect_new_events():
        await publisher.publish(event)

    return result
```

The publisher is passed into the service function through dependency injection — same pattern as UoW. The service function never knows what broker is behind the publisher.

---

## The Abstract Pattern

Same triple as repositories: abstract interface, concrete implementation, fake for tests. All in the same file.

### Event Publisher

```python
# infrastructure/streaming/publisher.py
import abc
from dataclasses import dataclass

class AbstractEventPublisher(abc.ABC):
    @abc.abstractmethod
    async def publish(self, event, subject: str | None = None) -> None:
        """Publish an event to the stream. Subject derived from event type if not provided."""
        ...


class EventPublisher(AbstractEventPublisher):
    """Concrete implementation — publishes to the stream broker."""
    def __init__(self, connection) -> None:
        self._connection = connection

    async def publish(self, event, subject: str | None = None) -> None:
        subject = subject or self._subject_for(event)
        data = serialize_event(event)
        await self._connection.publish(subject, data)

    def _subject_for(self, event) -> str:
        # ordering.Allocated, fulfillment.Shipped, etc.
        module = type(event).__module__.split(".")[-1]
        return f"{module}.{type(event).__name__}"


class FakeEventPublisher(AbstractEventPublisher):
    """In-memory implementation — records published events for test assertions."""
    def __init__(self) -> None:
        self.published_events: list = []

    async def publish(self, event, subject: str | None = None) -> None:
        self.published_events.append(event)
```

### Event Subscriber

```python
# infrastructure/streaming/subscriber.py
import abc
from typing import Callable, Awaitable

class AbstractEventSubscriber(abc.ABC):
    @abc.abstractmethod
    async def subscribe(self, subject: str, handler: Callable, durable_name: str | None = None) -> None:
        """Subscribe to a subject. Durable name enables position tracking and replay."""
        ...

    @abc.abstractmethod
    async def start(self) -> None:
        """Start consuming messages from all subscriptions."""
        ...

    @abc.abstractmethod
    async def stop(self) -> None:
        """Stop consuming and clean up."""
        ...


class EventSubscriber(AbstractEventSubscriber):
    """Concrete implementation — subscribes to the stream broker."""
    def __init__(self, connection) -> None:
        self._connection = connection
        self._subscriptions = []

    async def subscribe(self, subject, handler, durable_name=None):
        sub = await self._connection.subscribe(
            subject,
            durable=durable_name,
        )
        self._subscriptions.append((sub, handler))

    async def start(self) -> None:
        for sub, handler in self._subscriptions:
            # Each broker has its own consumption loop — this is broker-specific
            ...

    async def stop(self) -> None:
        for sub, _ in self._subscriptions:
            await sub.unsubscribe()


class FakeEventSubscriber(AbstractEventSubscriber):
    """In-memory implementation — records subscriptions for test verification."""
    def __init__(self) -> None:
        self.subscriptions: dict[str, list[Callable]] = {}

    async def subscribe(self, subject, handler, durable_name=None):
        self.subscriptions.setdefault(subject, []).append(handler)

    async def start(self) -> None:
        pass

    async def stop(self) -> None:
        pass

    async def simulate_event(self, subject: str, event) -> None:
        """Test helper — simulate an event arriving on a subject."""
        for handler in self.subscriptions.get(subject, []):
            await handler(event)
```

---

## Event Serialization

Events are frozen dataclasses. Serialization adds a type discriminator so the subscriber knows which class to deserialize into.

```python
# infrastructure/streaming/serialization.py
import json
import dataclasses

EVENT_REGISTRY: dict[str, type] = {}

def register_event(cls):
    """Decorator — registers an event class for deserialization."""
    EVENT_REGISTRY[f"{cls.__module__}.{cls.__qualname__}"] = cls
    return cls

def serialize_event(event) -> bytes:
    payload = {
        "event_type": f"{type(event).__module__}.{type(event).__qualname__}",
        "data": dataclasses.asdict(event),
    }
    return json.dumps(payload, default=str).encode()

def deserialize_event(raw: bytes):
    payload = json.loads(raw)
    cls = EVENT_REGISTRY[payload["event_type"]]
    return cls(**payload["data"])
```

Every shared event dataclass should be decorated with `@register_event`:

```python
# shared/events/ordering_events.py
from infrastructure.streaming.serialization import register_event
from dataclasses import dataclass

@register_event
@dataclass(frozen=True)
class Allocated:
    order_id: str
    batchref: str
    sku: str
    quantity: int
```

---

## Subject Naming

Subjects follow the pattern `{context}.{EventName}`:

```
ordering.Allocated
ordering.Deallocated
fulfillment.Shipped
fulfillment.PickListCreated
```

Streams are organized per context. A module subscribes with wildcards to consume all events from a context:

```python
# Subscribe to all ordering events
await subscriber.subscribe("ordering.*", handler, durable_name="fulfillment-ordering")

# Subscribe to a specific event
await subscriber.subscribe("ordering.Allocated", handler, durable_name="fulfillment-allocated")
```

---

## Event Handlers

Event handlers are service-layer functions in the subscribing module. Each handler receives an event and its own UoW — an independent transaction from whatever published the event.

```python
# modules/fulfillment/handlers/ordering_events.py
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

Handlers are registered during module startup:

```python
# modules/fulfillment/__init__.py
async def register(app, publisher, subscriber, uow_factory):
    from modules.fulfillment.handlers import ordering_events

    async def on_allocated(raw_event):
        event = deserialize_event(raw_event)
        uow = uow_factory()
        await ordering_events.handle_allocated(event, uow)

    await subscriber.subscribe("ordering.Allocated", on_allocated, durable_name="fulfillment-allocated")
```

---

## Replay for New Modules

When a new module is deployed, it needs to catch up on historical events. Persistent streams make this simple:

1. Set the durable consumer's deliver policy to "all" (replay from the beginning)
2. The subscriber processes every historical event in order
3. After catching up to the present, it continues consuming new events in real time
4. The broker tracks the consumer's position — if it restarts, it picks up where it left off

This is why event persistence matters. Without it, a new module has zero knowledge of what happened before it was deployed. With it, the module replays history and builds its state from events.

---

## Transaction Boundary

```
Service function:
  async with uow:           ← transaction opens
      domain logic           ← aggregates raise events to self.events
  ← UoW commits here        ← if exception, rolls back — no events published

  for event in uow.collect_new_events():
      await publisher.publish(event)   ← published ONLY after successful commit
```

If the UoW rolls back (domain exception, database error), the loop never runs. No events are published for failed operations.

Event handlers in other modules run in their own UoW. If a handler fails, it does not affect the original operation — the order was already placed, the allocation already committed. The handler can retry independently.

---

## Rules

| Rule | Why |
|---|---|
| Events are past-tense frozen dataclasses | They describe what happened — immutable facts |
| Events published only after successful commit | Prevents broadcasting something that was rolled back |
| Each event handler gets its own UoW | Independent transactions — handler failure doesn't affect the publisher |
| Domain never imports the broker | Domain is pure Python — the publisher is injected at the service layer |
| One publisher per event type | The module that owns the domain action publishes the event |
| Zero or many subscribers per event | Any module can subscribe — the publisher doesn't know who's listening |
| Subjects follow `{context}.{EventName}` | Predictable, discoverable, supports wildcard subscriptions |
| Shared event schemas are the inter-module contract | Frozen dataclasses in `shared/events/` — no logic, just data shapes |
| Version events by adding fields with defaults | Never remove fields — existing consumers depend on them |
