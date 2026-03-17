# Designing Fakes

Fakes are real Python objects that implement the same abstract interface as their production counterparts, but use in-memory data structures instead of databases or external services. They are the foundation of the testing strategy — every service-layer test runs against fakes.

---

## Why Fakes, Not Mocks

Mocks (`unittest.mock.patch`, `MagicMock`) record calls and replay canned responses. They test that your code called certain methods with certain arguments. They don't test that your code *works*.

A `MagicMock` repository will happily return whatever you tell it to, even if that response is impossible in real life. A fake repository actually stores and retrieves objects — it behaves like a real collection. If your service tries to get an object that was never added, the fake returns `None`, just like the real repository would.

Fakes test behavior. Mocks test wiring. We care about behavior.

---

## Fakes Live Alongside Their Real Counterparts

Every abstract class has its concrete implementation and its fake in the **same file**. This ensures they stay in sync — when you add a method to the abstract, both the real and fake implementations are right there demanding to be updated.

```
infrastructure/repositories/ordering/
    batch_repository.py    # AbstractBatchRepository + BatchRepository + FakeBatchRepository

unit_of_work/
    abstract.py            # AbstractUnitOfWork
    concrete.py            # UnitOfWork
    fake.py                # FakeUnitOfWork
```

For the UoW, the fake lives in its own file because it wires together multiple fake repositories and is conceptually distinct from the concrete UoW.

---

## Fake Repository

```python
# infrastructure/repositories/ordering/batch_repository.py
import abc
from domain.ordering.model import Batch

class AbstractBatchRepository(abc.ABC):
    def __init__(self) -> None:
        self.seen: set[Batch] = set()

    async def get(self, reference: str) -> Batch | None:
        batch = await self._get(reference)
        if batch:
            self.seen.add(batch)
        return batch

    async def get_by_sku(self, sku: str) -> list[Batch]:
        batches = await self._get_by_sku(sku)
        self.seen.update(batches)
        return batches

    async def add(self, batch: Batch) -> None:
        self.seen.add(batch)
        await self._add(batch)

    @abc.abstractmethod
    async def _get(self, reference: str) -> Batch | None: ...

    @abc.abstractmethod
    async def _get_by_sku(self, sku: str) -> list[Batch]: ...

    @abc.abstractmethod
    async def _add(self, batch: Batch) -> None: ...


class BatchRepository(AbstractBatchRepository):
    """Real implementation — talks to the database."""
    def __init__(self, conn) -> None:
        super().__init__()
        self.conn = conn

    async def _get(self, reference: str) -> Batch | None:
        # SQL query, _to_domain translation
        ...

    async def _get_by_sku(self, sku: str) -> list[Batch]:
        # SQL query, _to_domain translation
        ...

    async def _add(self, batch: Batch) -> None:
        # SQL insert
        ...


class FakeBatchRepository(AbstractBatchRepository):
    """In-memory implementation — used in tests."""
    def __init__(self, batches: list[Batch] | None = None) -> None:
        super().__init__()
        self._store: dict[str, Batch] = {b.reference: b for b in (batches or [])}

    async def _get(self, reference: str) -> Batch | None:
        return self._store.get(reference)

    async def _get_by_sku(self, sku: str) -> list[Batch]:
        return [b for b in self._store.values() if b.sku == sku]

    async def _add(self, batch: Batch) -> None:
        self._store[batch.reference] = batch
```

### Rules for fake repositories

- **Implement the full abstract interface.** Every abstract method gets a real implementation. No `pass`, no `raise NotImplementedError`.
- **Use simple data structures.** Dicts keyed by identity for `get`, list filtering for queries. Don't over-engineer the fake.
- **Mirror real behavior.** If the real repo returns `None` for missing items, the fake does too. If the real repo returns an empty list for no matches, the fake does too.
- **Accept initial data in the constructor.** `FakeBatchRepository(batches=[...])` lets tests set up state cleanly.

---

## Fake Unit of Work

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

### Key design decisions

- **`committed` flag** — tests assert `uow.committed is True` after successful operations and `uow.committed is False` after failures. This verifies the service layer correctly uses the UoW lifecycle.
- **No auto-commit in `__aexit__`** — the fake deliberately does NOT auto-commit on clean exit the way the real UoW does. This is intentional: the real UoW's `__aexit__` calls `commit()`, which sets `self.committed = True` through the fake's `commit()` method. The abstract base class handles this.
- **Wires all fake repositories** — when you add a new repository to `AbstractUnitOfWork`, add the corresponding fake to `FakeUnitOfWork`.

---

## Fake External Clients

For third-party services (email, payment, SMS), use the same pattern: abstract interface + real client + fake client, all in the same file.

```python
# infrastructure/gateways/email_client.py
import abc
from dataclasses import dataclass

@dataclass
class EmailPayload:
    to: str
    subject: str
    body: str

class AbstractEmailClient(abc.ABC):
    @abc.abstractmethod
    async def send(self, payload: EmailPayload) -> None: ...


class EmailClient(AbstractEmailClient):
    """Real implementation — calls SMTP/API."""
    def __init__(self, api_key: str):
        self._api_key = api_key

    async def send(self, payload: EmailPayload) -> None:
        # Real SMTP/API call
        ...


class FakeEmailClient(AbstractEmailClient):
    """In-memory implementation — records sent messages for test assertions."""
    def __init__(self) -> None:
        self.sent: list[EmailPayload] = []

    async def send(self, payload: EmailPayload) -> None:
        self.sent.append(payload)
```

In tests:
```python
async def test_order_confirmation_sends_email():
    uow = FakeUnitOfWork()
    email_client = FakeEmailClient()
    await create_order("order-1", "customer@example.com", uow, email_client)

    assert len(email_client.sent) == 1
    assert email_client.sent[0].to == "customer@example.com"
    assert "order-1" in email_client.sent[0].subject
```

The fake records what was sent. The test asserts on the recorded data. No mock needed — the fake is a real object with real state.

---

## When to Create a New Fake

Create a fake whenever you introduce a new abstract interface that the service layer depends on. The rule is simple: if the service function receives it as a parameter (directly or through UoW), it needs a fake for testing.

### Adding a new repository

1. Define `AbstractXRepository` with abstract methods
2. Implement `XRepository` (real, SQL-backed)
3. Implement `FakeXRepository` (in-memory) — in the same file
4. Add `x_items: AbstractXRepository` to `AbstractUnitOfWork`
5. Wire `FakeXRepository` into `FakeUnitOfWork`
6. Wire `XRepository` into `UnitOfWork`

### Adding a new external client

1. Define `AbstractXClient` with abstract methods
2. Implement `XClient` (real, API-backed)
3. Implement `FakeXClient` (in-memory recorder) — in the same file
4. Pass the client to service functions as a parameter
5. In tests, create `FakeXClient()` and pass it in
