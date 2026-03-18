# Module Structure — Sellable Units

---

## Purpose

A module is a bounded context packaged as an independently deployable, sellable unit. A customer might buy one module or several. Modules work alone or together — the code doesn't change, only the deployment configuration.

Modules share nothing at the domain level. No cross-module imports of domain objects, service functions, or repositories. The only shared thing is event schemas — frozen dataclasses that define the inter-module contract.

---

## Module Directory Layout

```
src/
  modules/
    ordering/
      domain/
        model.py           # Entities, Value Objects, Aggregates
        events.py           # Domain events (internal use — re-exports from shared/)
        commands.py         # Commands (if using command pattern internally)
        exceptions.py       # Domain exceptions
        services.py         # Domain services (stateless pure logic)
      service/
        allocation.py       # Use case functions — primitives in, primitives out
      infrastructure/
        repositories/
          batch_repository.py   # Abstract + Concrete + Fake
        schema/
          ordering.py           # SQLAlchemy Table definitions
      handlers/
        external_events.py  # Handlers for events FROM OTHER modules
      entrypoints/
        routers/
          allocation.py     # Thin routers
        schemas/
          allocation.py     # Pydantic request/response models
      __init__.py           # register(app, publisher, subscriber, uow_factory)

    fulfillment/
      domain/
        model.py
        events.py
        exceptions.py
      service/
        shipping.py
      infrastructure/
        repositories/
          shipment_repository.py
        schema/
          fulfillment.py
      handlers/
        external_events.py  # Handles ordering.Allocated, etc.
      entrypoints/
        routers/
          shipping.py
        schemas/
          shipping.py
      __init__.py

  shared/
    events/
      ordering_events.py    # Allocated, Deallocated — published by ordering
      fulfillment_events.py # Shipped, PickListCreated — published by fulfillment

  infrastructure/
    streaming/
      publisher.py          # AbstractEventPublisher + EventPublisher + FakeEventPublisher
      subscriber.py         # AbstractEventSubscriber + EventSubscriber + FakeEventSubscriber
      serialization.py      # serialize_event, deserialize_event, register_event
    database/
      connection.py         # create_engine
    postgresql/
      settings.py           # PostgresSettings

  unit_of_work/
    abstract.py             # AbstractUnitOfWork (with collect_new_events)
    concrete.py             # UnitOfWork
    fake.py                 # FakeUnitOfWork

  entrypoints/
    app.py                  # FastAPI app — loads enabled modules
    dependencies.py         # Dependency injection factories

  config.py                 # Settings, ENABLED_MODULES
```

---

## Shared Event Schemas

The `shared/events/` directory is the inter-module contract. It contains only frozen dataclasses — no logic, no imports from any module.

```python
# shared/events/ordering_events.py
from dataclasses import dataclass
from infrastructure.streaming.serialization import register_event

@register_event
@dataclass(frozen=True)
class Allocated:
    order_id: str
    batchref: str
    sku: str
    quantity: int

@register_event
@dataclass(frozen=True)
class Deallocated:
    order_id: str
    sku: str
    quantity: int
```

Each module's `domain/events.py` can re-export from here for internal use:

```python
# modules/ordering/domain/events.py
from shared.events.ordering_events import Allocated, Deallocated
```

When another module consumes these events, it imports directly from `shared/events/`:

```python
# modules/fulfillment/handlers/external_events.py
from shared.events.ordering_events import Allocated

async def handle_allocated(event: Allocated, uow: AbstractUnitOfWork) -> None:
    async with uow:
        pick = PickList(order_id=event.order_id, sku=event.sku, quantity=event.quantity)
        await uow.pick_lists.add(pick)
```

### Versioning events

Never remove fields from a shared event. Add new fields with defaults:

```python
@register_event
@dataclass(frozen=True)
class Allocated:
    order_id: str
    batchref: str
    sku: str
    quantity: int
    warehouse_id: str = ""   # Added in v2 — existing consumers ignore it
```

---

## Module Independence Rules

| Allowed | Forbidden |
|---|---|
| Import from `shared/events/` | Import from another module's `domain/` |
| Consume events from other modules via `handlers/` | Call another module's service functions directly |
| Each module has its own UoW and repositories | Share a UoW across modules |
| Each module has its own database schema (namespace) | Query another module's tables |
| Deploy one module or all modules | Require all modules to be present |

If you find yourself wanting to import from another module's domain, that's a signal: either the modules should be merged (they're the same bounded context) or you need an event.

---

## Module Registration

Each module exposes a `register()` function that wires it into the application:

```python
# modules/ordering/__init__.py
from fastapi import FastAPI
from infrastructure.streaming.publisher import AbstractEventPublisher
from infrastructure.streaming.subscriber import AbstractEventSubscriber
from infrastructure.streaming.serialization import deserialize_event

async def register(
    app: FastAPI,
    publisher: AbstractEventPublisher,
    subscriber: AbstractEventSubscriber,
    uow_factory,
) -> None:
    # 1. Add routers
    from modules.ordering.entrypoints.routers import allocation
    app.include_router(allocation.router, prefix="/api/v1")

    # 2. Subscribe to events from other modules (if any)
    from modules.ordering.handlers import external_events
    # ordering might listen to fulfillment.Shipped to update order status
    async def on_shipped(raw):
        event = deserialize_event(raw)
        uow = uow_factory()
        await external_events.handle_shipped(event, uow)

    await subscriber.subscribe("fulfillment.Shipped", on_shipped, durable_name="ordering-shipped")
```

---

## Loading Enabled Modules

The application entrypoint loads only the modules that are enabled:

```python
# entrypoints/app.py
import importlib
from contextlib import asynccontextmanager
from fastapi import FastAPI
from config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Infrastructure setup
    app.state.engine = create_engine(settings.postgres)
    app.state.publisher = create_publisher(settings.streaming)
    app.state.subscriber = create_subscriber(settings.streaming)

    # Register enabled modules
    for module_name in settings.ENABLED_MODULES:
        module = importlib.import_module(f"modules.{module_name}")
        await module.register(
            app=app,
            publisher=app.state.publisher,
            subscriber=app.state.subscriber,
            uow_factory=lambda: UnitOfWork(engine=app.state.engine),
        )

    # Start consuming events
    await app.state.subscriber.start()

    yield

    # Shutdown
    await app.state.subscriber.stop()
    await app.state.engine.dispose()

app = FastAPI(lifespan=lifespan)
```

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    ENABLED_MODULES: list[str] = ["ordering", "fulfillment"]
    # ...
```

If a customer only purchased the ordering module: `ENABLED_MODULES=["ordering"]`. No code changes — the fulfillment module simply isn't loaded.

---

## Single Process vs Microservices

The same code supports both deployment models. The difference is configuration.

**Single process** (all modules in one app):
```env
ENABLED_MODULES=["ordering", "fulfillment", "notifications"]
```

One FastAPI app, one container. All modules share the same NATS connection but communicate through events, not function calls.

**Microservices** (each module in its own app):
```env
# ordering-service
ENABLED_MODULES=["ordering"]

# fulfillment-service
ENABLED_MODULES=["fulfillment"]
```

Each module is its own container, its own FastAPI app. They connect to the same NATS cluster. Events flow between services exactly the same way they flow within a single process.

The code is identical. Only the deployment configuration changes.

---

## Replaying History for New Modules

When a new module is deployed for the first time, it needs to build its state from historical events.

1. The module's event handlers use durable consumers with deliver policy set to replay from the beginning
2. The subscriber processes every historical event in order
3. After catching up, it continues consuming new events in real time
4. On restart, the durable consumer picks up from its last acknowledged position

This is the primary reason for using a persistent stream. Without persistence, deploying a new module requires writing custom migration scripts to backfill state. With persistence, the module bootstraps itself by replaying the event stream.

---

## Rules

- **One module = one bounded context.** If two modules share the same aggregate, they're probably one module.
- **Modules communicate through events only.** No direct function calls, no shared repositories, no shared database tables.
- **Shared event schemas are the contract.** Treat them like a public API — additive changes only, never breaking changes.
- **Each module owns its database schema.** Use schema namespacing (e.g., `ordering.batches`, `fulfillment.shipments`) to keep data isolated.
- **Module registration is the only integration point.** The `register()` function is the complete interface between a module and the application.
