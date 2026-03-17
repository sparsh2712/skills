# Interaction Layer

---

## Purpose

The interaction layer translates between the outside world's protocol and the application's service layer. It is intentionally thin — no business logic, no repository access, no domain object construction.

The same service layer functions are callable from an HTTP router, an MCP tool handler, or a CLI command without modification. Each is a different adapter over the same application core.

---

## App Setup

```python
# entrypoints/app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from infrastructure.database.connection import create_engine
from infrastructure.email.factory import get_email_client
from middleware.error_handler import ErrorHandlerMiddleware
from entrypoints.routers import batches, orders
from config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.engine = create_engine(settings.postgres)
    app.state.email = get_email_client(settings.email)
    yield
    await app.state.engine.dispose()

app = FastAPI(lifespan=lifespan)
app.add_middleware(ErrorHandlerMiddleware)
app.include_router(batches.router, prefix="/api/v1")
app.include_router(orders.router, prefix="/api/v1")
```

---

## Dependencies

The concrete UoW is constructed here — the service layer only ever receives `AbstractUnitOfWork`:

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

---

## Thin Router

```python
# entrypoints/routers/allocation.py
from fastapi import APIRouter, Depends, status
from entrypoints.schemas.allocation import AllocateRequest, AllocateResponse, BatchCreateRequest
from entrypoints.dependencies import get_uow
from unit_of_work.abstract import AbstractUnitOfWork
from service.ordering import allocation as allocation_service

router = APIRouter(prefix="/allocations", tags=["allocations"])

@router.post("/batches", status_code=status.HTTP_201_CREATED)
async def add_batch(
    body: BatchCreateRequest,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> dict:
    ref = await allocation_service.create_batch(body.reference, body.sku, body.quantity, body.eta, uow)
    return {"reference": ref}

@router.post("", status_code=status.HTTP_201_CREATED)
async def allocate(
    body: AllocateRequest,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> AllocateResponse:
    result = await allocation_service.allocate(body.order_id, body.sku, body.quantity, uow)
    return AllocateResponse(batchref=result.batchref)

@router.delete("/{order_id}", status_code=status.HTTP_204_NO_CONTENT)
async def deallocate(
    order_id: str,
    sku: str,
    quantity: int,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> None:
    await allocation_service.deallocate(order_id=order_id, sku=sku, quantity=quantity, uow=uow)
```

Routers are the only place domain exceptions become HTTP errors. The domain and service layer never import from the web framework.

---

## Pydantic Schemas (Separate from Domain)

```python
# entrypoints/schemas/allocation.py
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class BatchCreateRequest(BaseModel):
    reference: str
    sku: str
    quantity: int
    eta: Optional[datetime] = None
    model_config = {"extra": "forbid"}

class AllocateRequest(BaseModel):
    order_id: str
    sku: str
    quantity: int
    model_config = {"extra": "forbid"}

class AllocateResponse(BaseModel):
    batchref: str
```

Schemas live in `entrypoints/schemas/` — never in the domain layer.

---

## Error Handler Middleware

The only place domain exceptions become HTTP status codes:

```python
# middleware/error_handler.py
from fastapi import Request
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from domain.ordering.exceptions import InvalidSkuError, OutOfStockError, BatchNotFoundError

EXCEPTION_STATUS_MAP = {
    InvalidSkuError: 400,
    OutOfStockError: 400,
    BatchNotFoundError: 404,
}

class ErrorHandlerMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        try:
            return await call_next(request)
        except tuple(EXCEPTION_STATUS_MAP) as exc:
            return JSONResponse(
                status_code=EXCEPTION_STATUS_MAP[type(exc)],
                content={"detail": str(exc)},
            )
        except Exception:
            return JSONResponse(status_code=500, content={"detail": "Internal server error"})
```

---

## CQRS Read Endpoints

For read-heavy endpoints, bypass the domain model entirely with a direct connection and a read query:

```python
# entrypoints/routers/batches.py
from infrastructure.repositories.ordering.read.batch_availability import get_batch_availability

@router.get("/batches/{sku}/availability")
async def batch_availability(
    sku: str,
    engine: AsyncEngine = Depends(get_engine),
) -> list[dict]:
    async with engine.connect() as conn:
        rows = await get_batch_availability(conn, sku)
    return [vars(row) for row in rows]
```

---

## HTTP Status Code Reference

| Scenario | Code |
|---|---|
| Successful read | 200 |
| Resource created | 201 |
| Async task accepted | 202 |
| Delete with no body | 204 |
| Business rule violation | 400 |
| Unauthenticated | 401 |
| Unauthorized | 403 |
| Resource not found | 404 |
| Duplicate / conflict | 409 |
| Validation error | 422 |
| Unexpected server error | 500 |
