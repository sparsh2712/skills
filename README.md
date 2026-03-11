# fastapi-backend-ddd

A Claude skill for building async Python backends using Domain-Driven Design. Teaches Claude the full 4-layer architecture, Repository pattern, Unit of Work, Aggregates, Bounded Contexts, and Ubiquitous Language — with complete working code examples and database-free unit testing via `FakeUnitOfWork`.

## What it does

When this skill is active, Claude will:

- Structure your project across four clean layers with strict dependency rules (Domain knows nothing about infrastructure)
- Name everything using **Ubiquitous Language** — the vocabulary of your domain, not generic technical terms
- Implement the **Repository pattern** with an abstract base, a concrete database implementation, and a fake for tests — all three, every time
- Wire the **Unit of Work** correctly — one commit point, repositories never touch transactions directly
- Keep the **domain layer completely free of infrastructure imports** — pure Python, instantly testable without a database

## Install

**Claude Code**
```bash
/plugin marketplace add sparsh2712/fastapi-backend-ddd
```

**Manual**
```bash
git clone git@github.com:sparsh2712/fastapi-ddd-skill.git
cp -r fastapi-backend-ddd ~/.claude/skills/
```

## What's inside

```
SKILL.md
resources/
  domain-model.md        # Entities, Value Objects, Aggregates, Domain Events
  repository-pattern.md  # Abstract + concrete + fake, CQRS read queries
  unit-of-work.md        # UoW pattern, FakeUnitOfWork, dependency injection
  service-layer.md       # Use case orchestration, primitives in/out, testing
  interaction-layer.md   # FastAPI routers, schemas, error middleware
  infrastructure.md      # Database connection, external service gateways
  bounded-context.md     # Ubiquitous Language, Anti-Corruption Layer
  complete-example.md    # Full end-to-end implementation (Batch/OrderLine)
```
