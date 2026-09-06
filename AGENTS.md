# URL Shortener agent guide

## Scope and sources of truth

- These instructions apply to the entire repository.
- Treat current code, tests, executable schemas, migrations, and configuration as authoritative once they exist.
- Use `docs/prd.md` for current product requirements and product-level constraints.
- Use `docs/architecture.md` for the currently implemented durable system structure, responsibilities, boundaries, and data flows. Do not describe proposed architecture there as though it already exists.
- Use `docs/schema.md` as a human-readable schema overview; executable migrations or schema definitions become authoritative when implementation exists.
- Treat `.plans/` and `.research/` as non-authoritative working material. Verify their current-state claims against code and durable documentation before relying on them.
- `.agents/skills/` contains reusable agent procedures. Keep them focused on durable workflows rather than application-specific implementation details.

## Repository conventions

- Use conventional uppercase names for standardized repository entry-point files such as `README.md` and `AGENTS.md`, and `SKILL.md` for skill entry points.
- Use lowercase kebab-case for ordinary Markdown documentation filenames and lowercase names for directories.
- Keep `.plans/` for proposed technical designs and implementation/execution plans.
- Keep `.research/` for audits, investigations, comparisons, and supporting research.
- Preserve empty tracked working directories with `.gitkeep` when necessary.

## Documentation workflow

- Update `docs/prd.md` when desired product behavior, scope, constraints, acceptance criteria, or product-level non-functional requirements change.
- For substantial planned work, use `.plans/` to capture proposed technical design, affected components and contracts, implementation steps, testing, and completion criteria. Small or routine changes do not require a plan unless explicitly requested.
- Record architecturally significant adopted decisions under `docs/adr/`. Do not create ADRs for routine implementation details, and do not rewrite accepted ADRs to match later decisions; supersede them when necessary.
- Update `docs/architecture.md` after implementation when a change alters durable system structure, responsibilities, boundaries, data flows, or other architectural behavior.
- Update `docs/schema.md` when the implemented data model changes materially; executable schemas and migrations remain authoritative.

For substantial changes, the usual flow is: clarify requirements, update the PRD if product requirements changed, create a design/implementation plan under `.plans/`, record significant adopted architectural decisions as ADRs, implement and verify the change, then update `docs/architecture.md` and other current-state documentation to match what was actually implemented.

Keep the roles distinct: the PRD describes desired product behavior, plans describe proposed changes, ADRs preserve decision rationale, and architecture documentation describes the current implemented system.

## Architecture constraints

The following adopted constraints should be preserved unless an explicit architectural change supersedes them:

- Every successfully created short code permanently maps to exactly one destination URI.
- Destination-to-short-code deduplication is best effort, not an authoritative uniqueness constraint.
- Generated and custom Base62 codes share one global namespace.
- Short URLs are permanent and immutable.
- Redirect reads are optimized around `short_code -> destination_uri`.
- Do not introduce stronger consistency, expiration, analytics, ownership, mutation, or multi-region behavior without first evaluating product-requirement and ADR impact.

These constraints describe adopted design decisions; do not assume corresponding runtime components exist until implementation evidence is present.

## Change guidance

- Keep HTTP handlers thin and place business flow in application/service-layer code when implementation begins.
- Keep infrastructure adapters behind clear boundaries for authentication, ID generation, shard routing, Redis, and PostgreSQL.
- Use bounded timeouts and retries for external dependencies; do not add unbounded retry loops to synchronous request paths.
- Prefer tests for invariants, routing, uniqueness, failure fallbacks, and concurrency behavior when implementation begins.
- Do not invent build, test, lint, or deployment commands before the repository actually defines them.
- After substantial changes, use the `maintain-docs` skill to evaluate living-documentation maintenance and the `maintain-adrs` skill to evaluate ADR maintenance.
- Use the `maintain-agents` skill when durable agent instructions may have changed; do not update `AGENTS.md` merely to record implementation history.
- Keep `README.md`, `AGENTS.md`, `docs/prd.md`, `docs/architecture.md`, `docs/schema.md`, and ADRs aligned with durable changes.
- After making changes, provide a suggested Conventional Commit message using `<type>[optional scope]: <description>`.
