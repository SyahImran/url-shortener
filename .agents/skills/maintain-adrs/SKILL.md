---
name: maintain-adrs
description: Evaluate substantial repository changes for architecturally significant decisions and maintain docs/adr/ when a new decision, supersession, or deprecation is warranted.
---

# Maintain Architecture Decision Records

Evaluate completed or planned repository changes against the ADR workflow in `docs/adr/README.md` and maintain the repository's architectural decision history only when warranted.

A successful use of this skill may result in no ADR changes.

## Sources and scope

Use this repository structure as the working convention:

- `docs/adr/README.md` defines ADR format, naming, statuses, and lifecycle.
- `docs/adr/` stores Architecture Decision Records and their lifecycle history.
- `docs/architecture.md` describes the current architecture.
- `.plans/` contains non-authoritative implementation or execution plans, including prompts intended to guide an agent through a change.
- `.research/` contains non-authoritative audits, investigations, comparisons, and supporting research.
- `AGENTS.md` contains durable instructions for coding agents.

Read applicable `AGENTS.md` instructions before changing files.

Inspect the current code, tests, configuration, contracts, documentation, relevant plans or research, and existing ADRs as needed to understand what was actually adopted. Prefer current executable or authoritative repository evidence over stale explanatory material.

## Significance test

Create or change an ADR only for an adopted decision that materially affects durable aspects of the system, such as:

- system structure or component boundaries;
- major dependencies or technology choices;
- data ownership or persistence strategy;
- public, cross-component, or cross-repository interfaces and contracts;
- communication or integration patterns;
- deployment or infrastructure architecture;
- security or trust boundaries;
- important operational characteristics;
- constraints that materially shape future implementation choices.

Ask whether future contributors or agents would reasonably need to know why this architectural choice was made.

Do not create ADRs for routine implementation choices, ordinary bug fixes, small refactors, naming or formatting changes, temporary workarounds, exploratory work, rejected alternatives, or plans that have not resulted in an adopted architectural decision.

When significance is marginal, prefer no ADR.

## Determine the action

Choose one outcome.

### No change

Use when no architecturally significant decision was adopted or existing ADRs already represent it accurately. Do not modify ADR files.

### Create

Create the next numbered ADR when a new architecturally significant decision has been adopted and is not already represented.

Follow `docs/adr/README.md` exactly. New ADRs begin as `Accepted`; do not create `Proposed` or `Rejected` ADRs.

### Supersede

When a new decision replaces an accepted ADR:

1. create a new ADR for the new decision;
2. mark the previous ADR `Superseded by ADR NNNN`;
3. preserve the previous ADR's Context, Decision, and Consequences.

Do not rewrite the old ADR to describe the new architecture.

### Deprecate

Mark an ADR `Deprecated` when its decision is no longer applicable or recommended and no single newer ADR replaces it. Preserve its historical content.

## Write concise ADRs

Use the format defined in `docs/adr/README.md`:

- **Context**: the forces, constraints, or circumstances that required the decision. Include alternatives only when they materially explain the choice.
- **Decision**: the adopted architectural choice, stated clearly and without incidental implementation detail.
- **Consequences**: meaningful tradeoffs, constraints, capabilities, operational implications, and future implementation effects.

Keep ADRs focused on the decision and rationale rather than duplicating plans, research logs, or architecture documentation.

## Keep documentation aligned

After creating, superseding, or deprecating an ADR, inspect `docs/architecture.md` and update it only if the current-state architecture would otherwise be inaccurate or materially incomplete.

Keep responsibilities distinct:

- `docs/adr/` records significant architectural decisions and rationale.
- `docs/architecture.md` describes the system as it exists now.
- `.plans/` holds implementation or execution plans.
- `.research/` holds audits, investigations, comparisons, and supporting research.
- `AGENTS.md` contains durable instructions agents need while changing the repository.

Do not copy ADR rationale into `AGENTS.md` unless the resulting decision creates a durable instruction or constraint agents must follow.

## Validate

Before finishing:

- verify the next ADR number and lowercase kebab-case filename;
- verify statuses follow `docs/adr/README.md`;
- verify supersession references are correct;
- confirm the decision is adopted and supported by repository evidence;
- confirm historical ADR content was preserved;
- confirm `docs/architecture.md` still describes the current architecture accurately;
- confirm no unnecessary ADR was created.

If no ADR maintenance is warranted, leave the ADR files unchanged and report that conclusion explicitly.
