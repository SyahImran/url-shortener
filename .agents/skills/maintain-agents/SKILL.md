---
name: maintain-agents
description: Audit and maintain AGENTS.md so it stays minimal, durable, accurate, and focused on repository-specific guidance that materially helps future coding agents.
---

# Maintain AGENTS.md

Audit `AGENTS.md` against the current repository and update it only where necessary.

Treat `AGENTS.md` as a compact instruction layer, not as architecture documentation, a repository tour, a changelog, or an inventory of current implementation details.

## Process

1. Read the existing `AGENTS.md`.
2. Inspect the repository as needed to verify its contents, including relevant code, tests, configuration, documentation, and development or CI tooling.
3. Verify repository-specific statements against current evidence.
4. Identify:
   - stale or incorrect instructions;
   - historical or completed-task information;
   - task-specific guidance;
   - directly observable implementation facts;
   - explanatory or architectural material better suited to other documentation;
   - redundant or overly verbose guidance;
   - missing durable constraints, workflows, or conventions that materially help future agents.
5. Update `AGENTS.md` only where doing so makes it more accurate, safer, clearer, or more concise.
6. Review the final file and justify every retained section against the criteria below.

## Retention test

For every statement or section, ask:

1. Would a capable coding agent discover this quickly from the repository when relevant?
2. Does stating it here materially reduce the risk of an incorrect, unsafe, or unnecessarily broad change?
3. Is it a durable instruction, invariant, workflow, convention, or non-obvious constraint?
4. Is `AGENTS.md` the best place for it rather than another repository document?

If information is easy to discover and does not materially prevent mistakes, remove it.

Accuracy alone is not sufficient reason to retain information.

## What belongs in AGENTS.md

Prefer concise, durable guidance such as:

- important repository-wide behavioral constraints;
- non-obvious architectural boundaries that constrain implementation;
- required development and verification procedures;
- dependency-management conventions;
- migration, generated-code, or schema-management rules;
- compatibility, security, or safety constraints;
- important invariants that are easy for an agent to violate;
- authoritative sources of truth;
- pointers to deeper documentation;
- instructions about areas that must not be modified casually.

Prefer rules over snapshots of current state.

For example, prefer:

> Runtime dependencies must be declared through the repository's established dependency-management mechanism.

over:

> Package X is currently listed in dependency file Y.

## What should usually be removed or moved elsewhere

Aggressively condense or remove:

- file-by-file repository tours;
- obvious entry-point descriptions;
- exhaustive lists of services, modules, routes, dependencies, formats, or features;
- exact counts of modes, components, tables, endpoints, or similar current-state details;
- changelog-style descriptions of completed work;
- temporary warnings that are no longer applicable;
- task-specific implementation details;
- facts directly visible by opening the referenced file;
- statements that merely confirm the current contents of another file;
- example diagnostic commands that are not established project verification;
- detailed execution flows better suited to architecture or design documentation;
- implementation explanations that do not constrain future work.

If valuable explanatory material does not belong in `AGENTS.md`, prefer pointing to an existing appropriate document.

Do not create new documentation merely to shorten `AGENTS.md` unless the invoking task explicitly asks for it.

## Concision standard

The goal is the smallest `AGENTS.md` that still prevents future agents from making repository-specific mistakes.

When multiple statements can be replaced by one durable rule without losing an important constraint, prefer the shorter rule.

Do not retain information merely because it is useful or accurate. It must be useful enough to justify being loaded into agent context repeatedly across unrelated future tasks.

Do not remove important non-obvious constraints merely to reduce length.

## Evidence and uncertainty

Base repository-specific guidance on evidence from the current repository.

Prefer current code and tests when determining actual behavior unless the repository explicitly defines another source as authoritative.

Do not invent conventions, workflows, commands, or architectural constraints.

If an existing instruction cannot be verified confidently, do not silently rewrite it as fact. Remove it if clearly obsolete, or report the uncertainty when it requires human judgment.

## Update rules

Do not update `AGENTS.md` merely because application code changed.

Update it only when the repository change affects durable instructions, workflows, architectural constraints, verification procedures, or conventions that future agents need.

Do not use `AGENTS.md` to record what was implemented, fixed, added, or removed unless the resulting state establishes a durable rule or constraint.

Do not modify application code, tests, dependencies, configuration, or unrelated documentation as part of this skill unless the invoking task explicitly requests those changes.

If `AGENTS.md` is already minimal, accurate, and sufficient under these criteria, leave it unchanged and report that no update was necessary.

## Completion

After the audit:

- summarize meaningful changes made to `AGENTS.md`;
- identify any content removed because it was stale, historical, directly observable, or better suited elsewhere;
- report any important uncertainty that could not be resolved from repository evidence;
- if no changes were necessary, say so explicitly.