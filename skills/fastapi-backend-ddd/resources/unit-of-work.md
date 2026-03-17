# Unit of Work Pattern

---

## Purpose

The Unit of Work (UoW) is the **single point of atomic transaction management**. It:

- Opens a database transaction
- Owns and exposes repositories for that transaction
- Is the only component that calls `commit()` or `rollback()`

The service layer accepts an abstract UoW and never knows whether it is talking to a real database or an in-memory fake. The concrete UoW is constructed at the interaction layer boundary (the `Depends()` factory in FastAPI) and passed down into service functions. Tests construct `FakeUnitOfWork` directly.

---

## Abstract Unit of Work (the interface)

```python
# unit_of_work/abstract.py
import abc
from infrastructure.repositories.ordering.batch_repository import AbstractBatchRepository

class AbstractUnitOfWork(abc.ABC):
    batches: AbstractBatchRepository

    async def __aenter__(self) -> "AbstractUnitOfWork":
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        if exc_type:
            await self.rollback()
        else:
            await self.commit()

    @abc.abstractmethod
    async def commit(self) -> None: ...

    @abc.abstractmethod
    async def rollback(self) -> None: ...
```

---

## Concrete Implementation

```python
# unit_of_work/concrete.py
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncConnection
from infrastructure.repositories.ordering.batch_repository import BatchRepository
from unit_of_work.abstract import AbstractUnitOfWork

class UnitOfWork(AbstractUnitOfWork):
    def __init__(self, engine: AsyncEngine) -> None:
        self._engine = engine

    async def __aenter__(self) -> "UnitOfWork":
        self._conn: AsyncConnection = await self._engine.connect()
        await self._conn.begin()
        self.batches = BatchRepository(self._conn)
        return await super().__aenter__()

    async def __aexit__(self, *args) -> None:
        await super().__aexit__(*args)
        await self._conn.close()

    async def commit(self) -> None:
        await self._conn.commit()

    async def rollback(self) -> None:
        await self._conn.rollback()
```

---

## Fake Unit of Work (for Unit Tests)

```python
# unit_of_work/fake.py
from infrastructure.repositories.ordering.batch_repository import FakeBatchRepository
from unit_of_work.abstract import AbstractUnitOfWork

class FakeUnitOfWork(AbstractUnitOfWork):
    def __init__(self) -> None:
        self.batches = FakeBatchRepository()
        self.committed = False
        self.rolled_back = False

    async def __aenter__(self) -> "FakeUnitOfWork":
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        if exc_type:
            await self.rollback()

    async def commit(self) -> None:
        self.committed = True

    async def rollback(self) -> None:
        self.rolled_back = True
```

`self.committed` lets tests assert that the service actually committed — a useful safeguard against silent failures.

---

## Usage in the Service Layer

```python
async def allocate(order_id: str, sku: str, quantity: int, uow: AbstractUnitOfWork) -> str:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        if not batches:
            raise InvalidSkuError(f"Invalid sku {sku}")
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        batchref = allocate_to_earliest_batch(batches, line)
        return batchref
```

`async with uow:` commits on clean exit and rolls back on any exception. The service never calls `commit()` directly.

---

## Dependency Injection in FastAPI

The concrete UoW is constructed once per request in the `Depends()` factory. This is the only place `UnitOfWork` is instantiated in production:

```python
# entrypoints/dependencies.py
from fastapi import Request, Depends
from sqlalchemy.ext.asyncio import AsyncEngine
from unit_of_work.concrete import UnitOfWork
from unit_of_work.abstract import AbstractUnitOfWork

def get_engine(request: Request) -> AsyncEngine:
    return request.app.state.engine

def get_uow(engine: AsyncEngine = Depends(get_engine)) -> AbstractUnitOfWork:
    return UnitOfWork(engine=engine)
```

The router passes it straight into the service call:

```python
@router.post("/allocations")
async def allocate(
    body: AllocateRequest,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> AllocateResponse:
    batchref = await allocation_service.allocate(body.order_id, body.sku, body.quantity, uow)
    return AllocateResponse(batchref=batchref)
```

In tests, `FakeUnitOfWork` is constructed directly in the test body and passed to the service the same way. The service function cannot tell the difference.

---

## Rules

- **Only UoW commits** — never `conn.commit()` in a repository or service function
- **UoW owns repositories** — instantiated inside `__aenter__`, not injected separately
- **One UoW per request** — never shared across requests
- **Always use as context manager** — `async with uow:` is the only correct usage
- **Concrete UoW constructed at interaction layer** — service layer only ever sees `AbstractUnitOfWork`
