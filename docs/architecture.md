# URL Shortener Architecture

## Status

Runtime implementation has not begun.

This document describes the currently implemented architecture only. The repository currently contains requirements, proposed design material, schema documentation, ADRs, and agent guidance, but no application runtime, database migrations, deployment configuration, or executable service architecture yet.

## Current repository state

The durable product requirements are defined in `docs/prd.md`.

The proposed initial technical design is captured in `.plans/initial-system-design.md`. It includes the intended service topology, API behavior, Snowflake/Base62 generation, PostgreSQL sharding, Redis caching, authentication, failure behavior, testing strategy, and implementation risks. Because that design has not yet been implemented, it is not current architecture.

Architecturally significant adopted decisions are recorded under `docs/adr/`. These decisions constrain the intended implementation, but this document should not present them as deployed behavior until corresponding code and infrastructure exist.

`docs/schema.md` is currently a human-readable proposed data-model overview. Once executable migrations or schema definitions exist, those become authoritative and this document should describe only the implemented storage architecture.

## Implemented components

None yet.

## Implemented runtime flows

None yet.

## Implemented persistence and caching

None yet.

## Implemented deployment topology

None yet.

## Documentation lifecycle

As implementation proceeds:

1. use `docs/prd.md` as the source for current product requirements;
2. use `.plans/` for proposed technical designs and implementation plans;
3. use `docs/adr/` to preserve rationale for significant adopted architectural decisions;
4. implement and verify the change against code, tests, migrations, and configuration; and
5. update this document after implementation so it reflects the architecture that actually exists.

Do not copy proposed architecture here before it is implemented. If implementation differs from the original plan, describe the verified implementation here and leave the plan as historical working material.
