# Infrastructure Layer

---

## Design Rules

- Infrastructure is a detail — it exists to serve the domain, not define it
- Settings via `BaseSettings` — every infra component reads from env vars through typed settings classes
- Repositories live in the infrastructure layer — they receive connections or clients as constructor arguments, never creating their own

---

## Database Connection

The connection layer creates an async engine at startup and stores it on application state. The Unit of Work acquires connections from it per request.

```python
# infrastructure/postgresql/settings.py
from pydantic_settings import BaseSettings

class PostgresSettings(BaseSettings):
    POSTGRES_HOST: str
    POSTGRES_PORT: int = 5432
    POSTGRES_USER: str
    POSTGRES_PASSWORD: str
    POSTGRES_DATABASE: str
    POSTGRES_POOL_SIZE: int = 5
    POSTGRES_MAX_OVERFLOW: int = 10

    model_config = {"env_file": ".env", "extra": "ignore"}
```

```python
# infrastructure/database/connection.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncEngine
from infrastructure.postgresql.settings import PostgresSettings

def create_engine(settings: PostgresSettings) -> AsyncEngine:
    url = (
        f"postgresql+asyncpg://{settings.POSTGRES_USER}:{settings.POSTGRES_PASSWORD}"
        f"@{settings.POSTGRES_HOST}:{settings.POSTGRES_PORT}/{settings.POSTGRES_DATABASE}"
    )
    return create_async_engine(
        url,
        pool_size=settings.POSTGRES_POOL_SIZE,
        max_overflow=settings.POSTGRES_MAX_OVERFLOW,
    )
```

---

## Database Schema Definition

Define database tables as SQLAlchemy `Table` objects in the infrastructure layer — one module per bounded context. The domain model never sees these. Repositories use them internally instead of raw SQL strings.

```python
# infrastructure/schema/ordering.py
from sqlalchemy import Table, Column, Integer, String, DateTime, ForeignKey, MetaData

metadata = MetaData(schema="ordering")

batches = Table(
    "batches", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("reference", String, unique=True, nullable=False),
    Column("sku", String, nullable=False),
    Column("purchased_quantity", Integer, nullable=False),
    Column("eta", DateTime, nullable=True),
)

order_lines = Table(
    "order_lines", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("order_id", String, nullable=False),
    Column("sku", String, nullable=False),
    Column("quantity", Integer, nullable=False),
)

allocations = Table(
    "allocations", metadata,
    Column("id", Integer, primary_key=True, autoincrement=True),
    Column("batch_id", Integer, ForeignKey("ordering.batches.id"), nullable=False),
    Column("orderline_id", Integer, ForeignKey("ordering.order_lines.id"), nullable=False),
)
```

Each bounded context gets its own schema module and its own Postgres schema prefix. This keeps contexts isolated at the database level.

```
infrastructure/
  schema/
    ordering.py       # Tables for the ordering context
    fulfillment.py    # Tables for the fulfillment context
```

### Using Table objects in repositories

Repositories import the `Table` objects and use SQLAlchemy's expression API for simple CRUD — get by id, insert, update. Column names are checked at import time — a rename in the schema definition immediately breaks any repository referencing the old name. For complex queries involving joins or aggregations, use explicit SQL with `text()` — the intent is clearer and easier to optimize.

```python
# infrastructure/repositories/ordering/batch_repository.py
from sqlalchemy import select
from infrastructure.schema.ordering import batches, order_lines, allocations

class BatchRepository(AbstractBatchRepository):
    def __init__(self, conn: AsyncConnection) -> None:
        super().__init__()
        self.conn = conn

    async def _get(self, reference: str) -> Batch | None:
        result = await self.conn.execute(
            select(batches).where(batches.c.reference == reference)
        )
        row = result.mappings().one_or_none()
        if row is None:
            return None
        allocs = await self._fetch_allocations(reference)
        return self._to_domain(row, allocs)

    async def _add(self, batch: Batch) -> None:
        await self.conn.execute(
            batches.insert().values(
                reference=batch.reference,
                sku=batch.sku,
                purchased_quantity=batch._purchased_quantity,
                eta=batch.eta,
            )
        )

    async def _save(self, batch: Batch) -> None:
        await self.conn.execute(
            batches.update()
            .where(batches.c.reference == batch.reference)
            .values(purchased_quantity=batch._purchased_quantity, eta=batch.eta)
        )
```

The `_to_domain` method stays the same — it still manually constructs a domain object from the row. The domain model is unchanged.

### Why Table objects instead of ORM models

SQLAlchemy ORM models (`class Batch(Base)`) merge schema definition with Python class behavior. In DDD, the domain model is a separate pure-Python class — you don't want SQLAlchemy's `Base`, `Column`, or session machinery in it. `Table` objects define the schema without creating classes, so the domain stays clean and the infrastructure owns the mapping.

---

## Email

```python
# infrastructure/email/client.py
import typing
from dataclasses import dataclass
from pydantic import BaseModel

class EmailPayload(BaseModel):
    from_address: str
    to: list[str]
    cc: list[str] | None = None
    subject: str
    body_html: str
    body_text: str

@dataclass
class SendResult:
    success: bool
    message_id: str | None
    error: str | None

class EmailClientProtocol(typing.Protocol):
    async def send(self, email: EmailPayload) -> SendResult: ...
    async def send_bulk(self, emails: list[EmailPayload]) -> list[SendResult]: ...
```

---

## External Service Gateways

Any external dependency the domain needs is accessed through a gateway. The gateway is defined in the infrastructure layer and passed into services as a constructor argument or function parameter. The domain never calls external services directly.

```python
# infrastructure/gateways/inventory_gateway.py
import typing
from dataclasses import dataclass

@dataclass
class StockLevel:
    sku: str
    available: int
    location: str

class InventoryGatewayProtocol(typing.Protocol):
    async def get_stock_level(self, sku: str) -> StockLevel: ...
    async def reserve(self, sku: str, quantity: int) -> bool: ...
```

---

## Lifecycle Ownership

Infrastructure clients are created once at startup and stored on application state. They are disposed on shutdown.

```python
# entrypoints/app.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from infrastructure.database.connection import create_engine
from infrastructure.email.factory import get_email_client
from config import settings

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.engine = create_engine(settings.postgres)
    app.state.email = get_email_client(settings.email)
    yield
    await app.state.engine.dispose()

app = FastAPI(lifespan=lifespan)
```

Nothing else manages client creation or disposal.
