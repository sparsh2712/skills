---
name: fastapi-backend-ddd
description: Domain-Driven Design implementation guidelines for async Python backends. Covers the 4-layer architecture (Infrastructure → Domain → Service → Interaction), the Repository pattern, Unit of Work, Aggregates, Value Objects, Entities, Bounded Contexts, and Ubiquitous Language. Use this skill whenever the user is building or designing a Python backend — whether they explicitly mention DDD or not. Trigger on any of these signals: structuring a FastAPI project, separating business logic from database code, designing domain models, defining service layers, writing repositories, implementing unit of work, building aggregates or value objects, working with domain events, defining bounded contexts, asking how to organize backend code, refactoring a messy service into clean layers, or building any non-trivial backend where business rules matter. If the user says things like "build me a backend for X", "how should I structure this API", or "I want clean architecture for my FastAPI app" — use this skill.
---

# Domain-Driven Design — Async Python Backend

---

## Foundational Philosophy

**Domain model is the heart.** Infrastructure serves the domain — not the other way around. The domain layer has zero infrastructure imports.

**Ubiquitous Language**: Class names, method names, module names, and API routes use the language of the business domain. If the domain expert says "allocate stock", the method is `allocate()` — not `process_inventory_adjustment()`.

**Naming is documentation.** No docstrings. No inline comments unless a non-obvious constraint must be preserved. Names carry all semantic weight. Explicit over clever.

**Flat over DRY.** Explicit, repetitive code that fits in one context window is worth more than elegant abstractions requiring ten file traversals to understand one operation.

---

## The Four-Layer Architecture

```
┌─────────────────────────────────────────────┐
│  Interaction Layer                          │  HTTP routers, CLI adapters, MCP handlers
│                                             │  Thin — translate protocol to service calls
├─────────────────────────────────────────────┤
│  Service Layer                              │  Use case orchestrators
│                                             │  Primitives in, primitives out
│                                             │  Uses domain + infrastructure (repositories)
├─────────────────────────────────────────────┤
│  Domain Layer                               │  Entities, Value Objects, Aggregates
│                                             │  Domain events, exceptions, domain services
│                                             │  Zero infrastructure imports — pure Python
├─────────────────────────────────────────────┤
│  Infrastructure Layer                       │  Databases, repositories, external services
└─────────────────────────────────────────────┘
```

**Layer rules:**
- Interaction calls Service — never Infrastructure directly
- Service calls Domain + Infrastructure (repositories, gateways, event publisher)
- Domain calls nothing — pure Python
- Infrastructure calls nothing above it
- Events cross module boundaries only through the event stream — never through direct imports

---

## Four Core Tactical DDD Patterns

| Pattern | Role |
|---|---|
| **Domain Model** | Entities, Value Objects, Aggregates — pure Python, zero infra imports |
| **Repository** | Hides persistence behind a collection interface. Abstract base + concrete implementation + fake for tests. |
| **Unit of Work** | Wraps a database transaction, owns repositories, is the only thing that commits. |
| **Service Layer** | Use case orchestrators. Accept primitives + UoW, call domain, return primitives. |
---

## Project Structure

```
src/
  domain/
    {context}/
      model.py        # Entities, Value Objects, Aggregates — no infra imports
      events.py       # Domain events (frozen dataclasses)
      commands.py     # Commands (frozen dataclasses)
      exceptions.py   # Domain exceptions
      services.py     # Domain services (stateless pure logic)

  service/
    {context}/
      {use_case}.py   # One file per use case group — accepts UoW, returns primitives

  infrastructure/
    repositories/
      {context}/
        {aggregate}_repository.py  # Abstract + concrete + fake per aggregate root

  unit_of_work/
    abstract.py       # AbstractUnitOfWork
    concrete.py       # Concrete UoW — wraps database connection
    fake.py           # FakeUnitOfWork — in-memory, for unit tests

  entrypoints/
    app.py            # Application setup, lifespan, middleware
    dependencies.py   # Dependency injection factories
    routers/
      {context}.py    # Thin routers
    schemas/
      {context}.py    # Request/response schemas

config.py
```

### Multi-Module Structure (Sellable Units)

For products where each bounded context is an independently deployable, sellable module:

```
src/
  modules/
    {module}/
      domain/            # Same structure as above
      service/
      infrastructure/
      handlers/          # Event handlers for events FROM other modules
      entrypoints/       # Routers and schemas for this module
      __init__.py        # register(app, publisher, subscriber, uow_factory)

  shared/
    events/
      {context}_events.py  # Shared event schemas — the inter-module contract

  infrastructure/
    streaming/
      publisher.py       # AbstractEventPublisher + concrete + fake
      subscriber.py      # AbstractEventSubscriber + concrete + fake
      serialization.py   # Event serialization/deserialization

  unit_of_work/
  entrypoints/
  config.py              # ENABLED_MODULES list
```

See resources/module-structure.md for the full pattern.

---

## Quick Start Checklists

### Strategic Planning Phase (before code)
- [ ] Interview the user to identify business capabilities — use the domain discovery protocol (resources/domain-discovery.md)
- [ ] Classify Subdomains — Core Domain (full DDD), Supporting (moderate), Generic (buy/use library)
- [ ] Identify Bounded Contexts — where does the model diverge? Ask the user if unclear
- [ ] Map Context Relationships — how do contexts communicate? (Partnership, Customer-Supplier, ACL, etc.)
- [ ] Establish Ubiquitous Language — define canonical terms per context with anti-terms
- [ ] Document Boundary Decisions — record rationale, consequences, and revisit triggers

### New Domain Feature
- [ ] Identify Bounded Context — which module owns this?
- [ ] Name everything using Ubiquitous Language — domain expert vocabulary only
- [ ] Define Entities, Value Objects, Aggregates in `domain/{context}/model.py` — no infra imports
- [ ] Define domain events in `domain/{context}/events.py`
- [ ] Define domain exceptions in `domain/{context}/exceptions.py`
- [ ] Write repository — Abstract base + concrete implementation + fake
- [ ] Write service functions — accept UoW and primitives, return primitives
- [ ] Publish events after commit if other modules need to react
- [ ] Write request/response schemas
- [ ] Write thin router

### New Module (Sellable Unit)
- [ ] Define module boundary — one bounded context per module
- [ ] Create `modules/{name}/domain/events.py` for published events
- [ ] Add shared event schemas to `shared/events/{name}_events.py`
- [ ] Create `modules/{name}/handlers/external_events.py` for consumed events
- [ ] Create `modules/{name}/__init__.py` with `register(app, publisher, subscriber, uow_factory)`
- [ ] Add module name to `ENABLED_MODULES` config

### New API Endpoint
- [ ] Router validates request via schema
- [ ] Router calls service function, passing UoW from dependency injection
- [ ] Service uses `async with uow:` — never calls commit directly
- [ ] Domain enforces all business rules
- [ ] Router maps domain exceptions to HTTP errors
- [ ] Response schema is separate from domain objects

---

## Topic Guides

- Domain Discovery & Interview Protocol → resources/domain-discovery.md
- Domain Model → resources/domain-model.md
- Infrastructure Layer → resources/infrastructure.md
- Repository Pattern → resources/repository-pattern.md
- Unit of Work → resources/unit-of-work.md
- Service Layer → resources/service-layer.md
- Interaction Layer → resources/interaction-layer.md
- Bounded Context & Ubiquitous Language → resources/bounded-context.md
- Event Streaming → resources/event-streaming.md
- Module Structure (Sellable Units) → resources/module-structure.md
- Testing → resources/testing.md
- Complete Example → resources/complete-example.md

---

## Core Principles

1. Domain has no infrastructure imports — pure Python, fast unit tests
2. Only UoW commits — repositories never call `commit()` or `rollback()`
3. Repositories operate on Aggregate roots only — no repository for internal entities
4. Ubiquitous Language in code — names are the documentation
5. Flat over DRY — repetitive but readable beats abstract and compact
6. TDD — FakeUoW + FakeRepository + FakeEventPublisher enable fast unit tests without infrastructure
7. The concrete UoW is constructed at the interaction layer boundary — service layer only ever sees the abstract interface
8. Events are published only after successful commit — no events on rollback
9. Modules communicate through events, never through shared domain objects
