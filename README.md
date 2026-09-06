# URL Shortener

A high-scale URL shortening service designed for authenticated URL creation and public redirects.

The system is designed around permanent immutable short URLs, Base62 codes, Snowflake-style distributed ID generation, Redis caching, and application-level PostgreSQL sharding.

## Documentation

- [Architecture](docs/architecture.md) — requirements, API contracts, data model, high-level architecture, data flows, scaling, caching, sharding, failure behavior, and implementation conventions.
- [Schema](docs/schema.md) — human-readable persistent data model.
- [Architecture Decision Records](docs/adr/README.md) — durable architecturally significant decisions.

## Repository working areas

- `.plans/` — non-authoritative implementation and execution plans intended to guide changes or coding agents.
- `.research/` — non-authoritative audits, investigations, comparisons, and supporting research.

## Status

The repository currently contains the proposed system design. Runtime implementation has not yet begun.
