# URL Shortener

A high-scale URL shortening service intended for authenticated URL creation and public redirects.

Runtime implementation has not yet begun. The repository currently contains product requirements, proposed technical design, schema documentation, ADRs, and reusable agent guidance.

## Documentation

- [Product requirements](docs/prd.md) — current product behavior, scope, constraints, and acceptance criteria.
- [Architecture](docs/architecture.md) — currently implemented system architecture. At present, no runtime architecture exists.
- [Schema](docs/schema.md) — human-readable data-model overview; executable migrations or schemas become authoritative once implementation exists.
- [Architecture Decision Records](docs/adr/README.md) — durable architecturally significant decisions and rationale.

## Repository working areas

- `.plans/` — non-authoritative proposed technical designs and implementation/execution plans intended to guide changes or coding agents.
- `.research/` — non-authoritative audits, investigations, comparisons, and supporting research.
- `.agents/skills/` — reusable agent workflows copied from `my-agent-harness`.

The initial proposed system design is in `.plans/initial-system-design.md`.

## Documentation workflow

For substantial changes: clarify requirements, update `docs/prd.md` when product requirements change, create or update a plan under `.plans/`, record significant adopted architectural decisions as ADRs, implement and verify the change, then update `docs/architecture.md` and other current-state documentation to match the actual implementation.

## Status

Design phase. No application runtime, migrations, or deployment configuration have been implemented yet.
