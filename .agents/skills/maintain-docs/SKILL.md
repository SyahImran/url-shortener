---
name: maintain-docs
description: Keep living repository documentation aligned with current product requirements and implemented system behavior after substantial changes, while preserving the distinction between PRDs, architecture, schema, plans, research, and ADRs.
---

# Maintain Repository Documentation

Review substantial completed or planned changes and maintain the repository's living documentation where warranted.

A successful use of this skill may result in no documentation changes.

## Sources and scope

Use this repository structure as the working convention:

- `docs/prd.md` describes current product requirements: what the product should do and why.
- `docs/architecture.md` describes the currently implemented architecture, responsibilities, boundaries, and important system behavior.
- `docs/schema.md` provides a human-readable data-model overview when useful; executable schemas and migrations remain authoritative where they exist.
- `docs/adr/` records significant architectural decisions and their rationale.
- `.plans/` contains non-authoritative technical designs and implementation or execution plans, including prompts intended to guide an agent through a change.
- `.research/` contains non-authoritative audits, investigations, comparisons, and supporting research.
- `AGENTS.md` contains durable instructions for coding agents.

Read applicable `AGENTS.md` instructions before changing files.

Inspect current code, tests, configuration, contracts, schemas, migrations, documentation, relevant plans or research, and ADRs as needed. Prefer executable and authoritative repository evidence over stale explanatory material.

## Determine what changed

Classify the change before editing documentation.

### Product requirements

Update `docs/prd.md` when the desired product behavior changes materially, including:

- capabilities are added or removed;
- user-visible behavior or workflows change;
- product scope changes;
- important constraints change;
- acceptance criteria change;
- product-level non-functional requirements change.

Do not update the PRD merely because an implementation detail or technology choice changed without affecting product requirements.

Keep the PRD focused on the current desired product rather than implementation history.

### Current architecture

Update `docs/architecture.md` when the implemented system architecture changes materially, including:

- component responsibilities or boundaries;
- service interactions or data flows;
- persistence, caching, messaging, or integration patterns;
- public or cross-component contracts when architecturally relevant;
- deployment or infrastructure topology;
- security or trust boundaries;
- important runtime or operational behavior.

Do not describe proposed architecture as current architecture. For planned but unimplemented changes, keep the future-state design in `.plans/`.

Once implementation is complete and verified, ensure `docs/architecture.md` describes what actually exists, including deviations from the original plan.

### Human-readable schema

Update `docs/schema.md` when the implemented data model changes materially and the repository uses this document.

Keep it aligned with authoritative schemas and migrations. Do not make `docs/schema.md` override executable schema definitions.

### Architectural decisions

When the work introduces, replaces, or retires a significant architectural decision, use the `maintain-adrs` skill rather than embedding decision history in living documentation.

Keep the distinction clear:

- `architecture.md` says what the system does now;
- ADRs explain why significant architectural choices were made.

### Agent instructions

Do not put ordinary product, architecture, or implementation details into `AGENTS.md`.

If a change creates or alters a durable instruction about how agents must work in the repository, use the appropriate agent-guidance maintenance workflow separately.

## Respect working artifacts

Treat `.plans/` and `.research/` as supporting evidence, not current-state sources of truth.

A plan may describe an intended future state that was never implemented or that changed during implementation. Research may contain alternatives, hypotheses, or outdated findings.

Before promoting information from either area into durable documentation, verify it against the current repository.

Do not rewrite completed plans merely to make them match the final implementation. Preserve them as working or historical artifacts unless the repository explicitly uses another lifecycle.

## Avoid unnecessary documentation

Not every change requires documentation maintenance.

Usually make no documentation changes for:

- small bug fixes that restore already-documented behavior;
- formatting or naming changes with no durable semantic effect;
- internal refactors that preserve documented architecture and contracts;
- temporary debugging or exploratory work;
- implementation details too narrow to help future contributors.

When significance is marginal, prefer leaving durable documentation unchanged.

## Keep documents focused

Avoid copying the same information across PRDs, architecture docs, ADRs, and plans.

Use these boundaries:

- PRD: current product intent and requirements;
- `.plans/`: proposed technical design and execution steps;
- ADR: significant architectural decision and rationale;
- `architecture.md`: current implemented system structure and behavior;
- `schema.md`: current human-readable data model;
- `.research/`: investigation and supporting evidence.

Cross-reference another document when useful rather than duplicating its full rationale or contents.

## Validate

Before finishing:

- confirm each changed document still serves its intended role;
- verify PRD changes reflect product requirements rather than implementation detail;
- verify architecture documentation reflects implemented reality rather than a proposal;
- verify schema documentation agrees with authoritative schemas or migrations;
- verify significant architectural decisions were evaluated through `maintain-adrs`;
- verify plans and research were not treated as authoritative without validation;
- verify unrelated documentation was not changed unnecessarily;
- verify no important current-state documentation became stale because of the change.

If no living documentation maintenance is warranted, leave the documentation unchanged and report that conclusion explicitly.
