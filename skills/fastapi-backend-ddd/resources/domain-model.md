# Domain Model — Entities, Value Objects, Aggregates

---

## The Prime Directive: Persistence Ignorance

The domain layer has **zero imports from any infrastructure library**. It is plain Python. This is the structural guarantee that makes fast unit tests and infrastructure swappability possible.

```python
# domain/ordering/model.py — no infra imports
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from enum import Enum
from typing import Optional
from domain.ordering.events import Allocated, OrderConfirmed
from domain.ordering.exceptions import OutOfStockError, OrderAlreadyConfirmedError
```

---

## Entities

An Entity is defined by its **identity**, not its attributes. Two entities with the same data but different IDs are different objects. Entities are **mutable** and have a meaningful lifecycle.

```python
class Order:
    def __init__(self, order_id: str, customer_id: str) -> None:
        self.order_id = order_id
        self.customer_id = customer_id
        self.lines: list[OrderLine] = []
        self.status: OrderStatus = OrderStatus.DRAFT
        self.events: list = []

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Order):
            return False
        return self.order_id == other.order_id

    def __hash__(self) -> int:
        return hash(self.order_id)

    def add_line(self, line: OrderLine) -> None:
        if self.status != OrderStatus.DRAFT:
            raise OrderAlreadyConfirmedError(f"Cannot modify confirmed order {self.order_id}")
        self.lines.append(line)

    def confirm(self) -> None:
        if not self.lines:
            raise ValueError(f"Cannot confirm order {self.order_id} with no lines")
        self.status = OrderStatus.CONFIRMED
        self.events.append(OrderConfirmed(order_id=self.order_id, customer_id=self.customer_id))
```

---

## Value Objects

A Value Object has **no identity** — defined entirely by its attributes. Immutable. Two Value Objects with the same attributes are equal and interchangeable. Use frozen dataclasses.

```python
@dataclass(frozen=True)
class OrderLine:
    order_id: str
    sku: str
    quantity: int

    def __post_init__(self) -> None:
        if self.quantity <= 0:
            raise ValueError(f"Quantity must be positive: {self.quantity}")

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

    def __add__(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError(f"Cannot add {self.currency} and {other.currency}")
        return Money(amount=self.amount + other.amount, currency=self.currency)

    def __mul__(self, factor: Decimal) -> "Money":
        return Money(amount=self.amount * factor, currency=self.currency)
```

**Decision rule**: Ask "do we care which instance this is?" No → Value Object. Yes → Entity.

---

## Aggregates & Consistency Boundaries

An Aggregate is a **cluster of Entities and Value Objects** treated as a single unit for data changes. It has one **Aggregate Root** — the only object external code may hold a reference to.

```python
class Batch:  # Aggregate root
    def __init__(
        self,
        reference: str,
        sku: str,
        purchased_quantity: int,
        eta: Optional[datetime],
    ) -> None:
        self.reference = reference
        self.sku = sku
        self._purchased_quantity = purchased_quantity
        self.eta = eta
        self._allocations: set[OrderLine] = set()
        self.events: list = []

    @property
    def available_quantity(self) -> int:
        return self._purchased_quantity - sum(line.quantity for line in self._allocations)

    def can_allocate(self, line: OrderLine) -> bool:
        return self.sku == line.sku and self.available_quantity >= line.quantity

    def allocate(self, line: OrderLine) -> None:
        if not self.can_allocate(line):
            raise OutOfStockError(f"Cannot allocate {line.quantity} of {line.sku}")
        self._allocations.add(line)
        self.events.append(Allocated(
            order_id=line.order_id, sku=line.sku,
            quantity=line.quantity, batchref=self.reference,
        ))

    def deallocate(self, line: OrderLine) -> None:
        self._allocations.discard(line)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Batch):
            return False
        return self.reference == other.reference

    def __hash__(self) -> int:
        return hash(self.reference)

    def __gt__(self, other: "Batch") -> bool:
        if self.eta is None:
            return False
        if other.eta is None:
            return True
        return self.eta > other.eta
```

**Aggregate rules:**
- External objects hold references to the root only — never to internal members
- Only the root enforces consistency within the aggregate boundary
- One repository per Aggregate root — never a repository for an internal entity

---

## Domain Events

Past tense. Immutable frozen dataclasses. No infrastructure references.

```python
# domain/ordering/events.py
from dataclasses import dataclass

@dataclass(frozen=True)
class Allocated:
    order_id: str
    sku: str
    quantity: int
    batchref: str

@dataclass(frozen=True)
class Deallocated:
    order_id: str
    sku: str
    quantity: int

@dataclass(frozen=True)
class OrderConfirmed:
    order_id: str
    customer_id: str
```

Aggregates append events to `self.events`. What consumes those events is determined at the infrastructure boundary — the domain never knows.

After the UoW commits, `uow.collect_new_events()` drains these events from all touched aggregates. The service function then publishes them to the event stream for cross-module communication. See resources/event-streaming.md for the full pattern.

In multi-module systems, event dataclasses used for cross-module communication live in `shared/events/` as the inter-module contract. A module's `domain/events.py` can re-export from there. See resources/module-structure.md.

---

## Domain Commands

Imperative. Frozen dataclasses. Represent intent to perform an operation.

```python
# domain/ordering/commands.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass(frozen=True)
class Allocate:
    order_id: str
    sku: str
    quantity: int

@dataclass(frozen=True)
class CreateBatch:
    reference: str
    sku: str
    quantity: int
    eta: Optional[datetime]
```

---

## Domain Exceptions

Express business rule violations. Never raise HTTP errors from the domain.

```python
# domain/ordering/exceptions.py
class OutOfStockError(Exception):
    pass

class InvalidSkuError(Exception):
    pass

class OrderAlreadyConfirmedError(Exception):
    pass

class BatchNotFoundError(Exception):
    pass
```

---

## Domain Services

Stateless operations that belong in the domain but don't fit naturally on any Entity or Aggregate. No database access. Pure computation.

```python
# domain/ordering/services.py
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

## What Goes in the Domain Layer

| Allowed | Forbidden |
|---|---|
| Plain Python classes | Any database library import |
| `dataclasses` | Any web framework import |
| `decimal.Decimal` | Database column definitions |
| `enum.Enum` | Connection or session objects |
| Domain exceptions | HTTP status codes |
| Domain events (frozen dataclasses) | External service calls |
| Domain services (pure functions) | Any I/O |
