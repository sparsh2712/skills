# Complete Example — Stock Allocation System

This walkthrough shows how the individual patterns wire together into a working system. Each section references the resource file where the pattern is explained in full. Read those files for detailed guidance — this page shows the assembly.

---

## System Overview

A stock allocation service: batches of stock arrive from suppliers, and order lines are allocated to batches (earliest first). The domain enforces that you cannot allocate more than available stock.

**Bounded Context:** Ordering
**Aggregate Root:** Batch (owns OrderLine allocations internally)
**Key Use Cases:** create batch, allocate order line, deallocate order line

---

## File Layout

```
src/
  domain/ordering/
    model.py          # Batch (aggregate root), OrderLine (value object)
    events.py         # Allocated, Deallocated
    exceptions.py     # OutOfStockError, InvalidSkuError, BatchNotFoundError
    services.py       # allocate_to_earliest_batch (domain service)

  infrastructure/repositories/ordering/
    batch_repository.py  # AbstractBatchRepository, BatchRepository, FakeBatchRepository

  unit_of_work/
    abstract.py       # AbstractUnitOfWork
    concrete.py       # UnitOfWork
    fake.py           # FakeUnitOfWork

  service/ordering/
    allocation.py     # create_batch, allocate, deallocate

  entrypoints/
    app.py            # FastAPI app, lifespan
    dependencies.py   # get_engine, get_uow
    routers/
      allocation.py   # POST /allocations, POST /allocations/batches, DELETE
    schemas/
      allocation.py   # Pydantic request/response models

  middleware/
    error_handler.py  # Maps domain exceptions → HTTP status codes

tests/
  unit/
    domain/test_batch.py
    service/test_allocation.py
  integration/
    test_repository.py
```

---

## How the Layers Connect

### 1. Domain Layer (pure Python — see resources/domain-model.md)

`Batch` is the aggregate root. It enforces the invariant: cannot allocate more than available stock. `OrderLine` is a frozen value object. Domain events are appended to `self.events` on state changes.

The domain service `allocate_to_earliest_batch` sorts batches by ETA and allocates to the first one with capacity.

### 2. Infrastructure Layer (see resources/repository-pattern.md, resources/infrastructure.md)

`AbstractBatchRepository` defines the collection interface with a `seen` set for tracking touched aggregates. `BatchRepository` implements it with raw SQL and an explicit `_to_domain` translation. `FakeBatchRepository` implements it with an in-memory dict for tests.

### 3. Unit of Work (see resources/unit-of-work.md)

`AbstractUnitOfWork` exposes repositories and manages the transaction boundary. It auto-commits on clean exit and auto-rolls-back on exception. `UnitOfWork` wraps a real SQLAlchemy async connection. `FakeUnitOfWork` uses fake repositories and tracks `committed` for test assertions.

### 4. Service Layer (see resources/service-layer.md)

Service functions accept primitives + UoW, construct domain objects internally, delegate to the domain, and return primitives. The caller never touches domain classes.

```python
async def allocate(order_id: str, sku: str, quantity: int, uow: AbstractUnitOfWork) -> AllocationResult:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        if not batches:
            raise InvalidSkuError(f"Invalid sku {sku}")
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        batchref = allocate_to_earliest_batch(batches, line)
        return AllocationResult(batchref=batchref, sku=sku)
```

### 5. Interaction Layer (see resources/interaction-layer.md)

The router is thin — validates via Pydantic schema, calls the service function, returns a response. The concrete UoW is injected via `Depends(get_uow)`.

```python
@router.post("", status_code=status.HTTP_201_CREATED)
async def allocate(
    body: AllocateRequest,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> AllocateResponse:
    result = await allocation_service.allocate(body.order_id, body.sku, body.quantity, uow)
    return AllocateResponse(batchref=result.batchref)
```

Domain exceptions become HTTP errors in `ErrorHandlerMiddleware` — the only place that maps domain exceptions to status codes.

### 6. Tests (see resources/testing.md)

Domain tests hit `Batch` and `OrderLine` directly — pure Python, no fakes needed. Service tests use `FakeUnitOfWork` to verify orchestration and commit behavior. Integration tests use the real database to verify repository round-trips.

---

## Request Flow

```
HTTP POST /allocations {order_id, sku, quantity}
  │
  ├─ Router: validates via AllocateRequest schema
  ├─ Router: calls allocation_service.allocate(order_id, sku, qty, uow)
  │    │
  │    ├─ Service: async with uow  (opens transaction)
  │    ├─ Service: batches = await uow.batches.get_by_sku(sku)
  │    ├─ Service: line = OrderLine(...)
  │    ├─ Service: batchref = allocate_to_earliest_batch(batches, line)
  │    │    │
  │    │    ├─ Domain service: sorts batches, finds first with capacity
  │    │    └─ Domain: batch.allocate(line) — enforces invariant, appends event
  │    │
  │    ├─ Service: returns AllocationResult(batchref, sku)
  │    └─ UoW: auto-commits on clean exit
  │
  └─ Router: returns AllocateResponse {batchref}
```

---

## Key Wiring Decisions

| Decision | Where | Why |
|---|---|---|
| Concrete UoW constructed in `dependencies.py` | Interaction layer | Service layer only sees the abstract interface — swappable in tests |
| Repository `_to_domain` translates rows to domain objects | Infrastructure | Domain model stays persistence-ignorant |
| Domain exceptions mapped to HTTP in middleware | Interaction layer | Domain never imports web framework |
| `seen` set on repository | Infrastructure | Lets UoW inspect which aggregates were touched |
| Pydantic schemas separate from domain | Interaction layer | API contract can evolve independently from domain |
