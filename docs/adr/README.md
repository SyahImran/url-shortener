# Architecture Decision Records

This directory stores Architecture Decision Records (ADRs) using the lightweight Nygard format.

Use an ADR for an adopted decision that is architecturally significant: one that affects important structure, dependencies, interfaces, constraints, operational characteristics, or future implementation choices.

Do not use ADRs for routine implementation choices, temporary plans, exploratory work, or rejected alternatives.

## File naming

Use a four-digit sequence number followed by a short lowercase kebab-case title:

```text
0001-use-snowflake-base62-short-codes.md
0002-use-sharded-postgresql-for-authoritative-storage.md
```

Sequence numbers are never reused.

## Record format

```markdown
# ADR NNNN: Decision title

## Status

Accepted

## Context

What forces, constraints, or circumstances required the decision?

## Decision

What was decided?

## Consequences

What becomes easier, harder, constrained, or possible because of the decision?
```

Accepted ADRs are historical records. When a later decision replaces one, create a new ADR and mark the old record `Superseded by ADR NNNN`. Use `Deprecated` when a decision is no longer applicable but is not replaced by one specific ADR.
