# URL Shortener agent guide

## Scope and sources of truth

- These instructions apply to the entire repository.
- Treat current code, tests, executable schemas, and configuration as authoritative once they exist.
- Use `docs/architecture.md` for the intended durable system structure, responsibilities, constraints, and data flows.
- Use `docs/schema.md` as a human-readable schema overview; executable migrations or schema definitions become authoritative when implementation exists.
- Treat `.plans/` and `.research/` as non-authoritative working material. Verify their current-state claims against code and durable documentation before relying on them.

## Repository conventions

- Use conventional uppercase names for standardized repository entry-point files such as `README.md` and `AGENTS.md`.
- Use lowercase kebab-case for ordinary Markdown documentation filenames and lowercase names for directories.
- Keep `.plans/` for implementation/execution plans and `.research/` for audits, investigations, and comparisons.
- Preserve empty tracked working directories with `.gitkeep` when necessary.

## Architecture constraints

- Preserve the invariant that every successfully created short code permanently maps to exactly one destination URI.
- Treat destination-to-short-code deduplication as best effort, not as an authoritative uniqueness constraint.
- Keep generated and custom Base62 codes in one global namespace.
- Keep short URLs permanent and immutable unless an explicit architecture change supersedes that constraint.
- Keep redirect reads optimized around `short_code -> destination_uri`.
- Do not introduce stronger consistency, expiration, analytics, ownership, mutation, or multi-region behavior without updating the architecture and evaluating ADR impact.

## Documentation and decisions

- Keep `docs/architecture.md` focused on durable current structure, behavior, boundaries, trade-offs, and operational constraints rather than implementation history.
- Record architecturally significant adopted decisions under `docs/adr/` using the workflow documented there.
- Do not create ADRs for routine implementation details.
- Do not rewrite accepted ADRs to match later decisions; supersede them with a new ADR when necessary.
- After substantial architectural changes, evaluate whether relevant ADRs and `docs/architecture.md` need maintenance.

## Change guidance

- Keep HTTP handlers thin and place business flow in application/service-layer code when implementation begins.
- Keep infrastructure adapters behind clear boundaries for authentication, ID generation, shard routing, Redis, and PostgreSQL.
- Use bounded timeouts and retries for external dependencies; do not add unbounded retry loops to synchronous request paths.
- Prefer tests for invariants, routing, uniqueness, failure fallbacks, and concurrency behavior when implementation begins.
- Do not invent build, test, lint, or deployment commands before the repository actually defines them.
- Keep `README.md`, `AGENTS.md`, `docs/architecture.md`, `docs/schema.md`, and ADRs aligned with durable changes.
- After making changes, provide a suggested Conventional Commit message using `<type>[optional scope]: <description>`.
