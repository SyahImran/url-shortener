---
name: create-agents
description: Create or bootstrap an AGENTS.md for an existing repository by inspecting the repository's code, configuration, tests, documentation, and established conventions. Use when a repository does not yet have an AGENTS.md or when asked to generate one from the current repository.
---

# Create AGENTS.md

Create a concise, durable `AGENTS.md` that helps coding agents work correctly in an existing repository.

The repository itself is the primary source of truth. Do not invent conventions, architecture, commands, or constraints that cannot be supported by repository evidence.

## 1. Determine scope

Identify the repository root before inspecting it.

If invoked from a parent workspace containing multiple repositories:

* determine which repository or repositories the request applies to;
* inspect each target repository independently;
* create an `AGENTS.md` inside each target repository rather than combining unrelated repositories into one file;
* do not treat the parent workspace as a repository unless it actually has repository-level code or configuration that warrants its own instructions.

Before creating a new file, search upward and within the target repository for existing `AGENTS.md` files. Respect any applicable parent instructions.

## 2. Inspect the repository

Build a working understanding of the repository before writing `AGENTS.md`.

Inspect high-signal sources first:

* top-level directory structure;
* README and contributor documentation;
* package and dependency manifests;
* build configuration;
* test configuration and representative tests;
* linting and formatting configuration;
* CI/CD workflows;
* environment/configuration templates;
* database migrations and seeders when relevant;
* application entry points;
* major source directories;
* architecture or contract documentation;
* scripts and task runners;
* existing agent or AI-assistant instructions.

Do not exhaustively read generated files, dependency directories, build artifacts, vendored code, large datasets, or unrelated historical documentation.

Use targeted searches after the initial inspection to verify important claims.

## 3. Infer only established conventions

Derive instructions from repeated or explicit repository evidence.

Good candidates include:

* authoritative build, test, lint, and type-check commands;
* architectural boundaries;
* source-of-truth files;
* synchronization requirements between files or components;
* security boundaries;
* configuration conventions;
* database migration rules;
* API or schema contracts;
* testing expectations;
* generated-code boundaries;
* naming or structural conventions that are consistently enforced;
* important compatibility requirements.

Do not turn incidental implementation details into permanent rules.

Distinguish between:

* **durable constraints** — rules an agent should normally preserve;
* **current implementation details** — useful for understanding the code but likely to change;
* **historical context** — generally unsuitable for `AGENTS.md`.

Prefer durable constraints.

## 4. Verify commands

Before documenting a command, establish that it comes from repository evidence such as:

* `package.json`;
* `Makefile`;
* task runner configuration;
* CI workflows;
* README or contributor documentation;
* language-specific project configuration;
* existing scripts.

When practical, run safe read-only or validation commands to confirm them.

Do not invent conventional commands merely because they are common for the language or framework.

If different parts of the repository require different commands, state their working directories clearly.

## 5. Identify sources of truth

Where multiple files describe the same behavior, determine which should be treated as authoritative.

First respect source-of-truth relationships explicitly defined by the repository. For example, a repository may designate an API specification, schema, contract document, generated source, or other artifact as authoritative over an implementation that must conform to it.

Where no source is explicitly designated as authoritative, prefer, in roughly this order:

1. current executable code and tests;
2. active configuration and schemas;
3. current architecture or contract documentation;
4. README/contributor guidance;
5. historical plans, audits, or migration notes.

Call out important source-of-truth relationships when they prevent agents from making inconsistent changes.

Examples include:

* an enum mirrored between backend and frontend;
* configuration synchronized with model definitions;
* generated clients derived from an API specification;
* migrations paired with runtime seeders;
* documentation that defines an external contract.

Do not list every file in the repository.

## 6. Capture important boundaries

Look specifically for boundaries that an automated coding agent could accidentally violate, including:

* authentication and authorization;
* secret handling;
* filesystem containment;
* network or API trust boundaries;
* database ownership;
* migration compatibility;
* generated files;
* public API compatibility;
* cross-service contracts;
* storage abstractions;
* concurrency or transactional assumptions.

Include them only when supported by repository evidence.

## 7. Write for future changes

`AGENTS.md` should tell an agent how to work in the repository, not describe the repository line by line.

Prefer instructions such as:

> Keep processing modes synchronized between the model definition, pipeline configuration, client defaults, and associated tests.

over brittle descriptions such as:

> The application currently has three processing modes.

Prefer patterns and invariants over snapshots.

Avoid embedding:

* current branch names;
* commit hashes unless permanently significant;
* temporary migration state;
* recently completed implementation plans;
* issue-specific instructions;
* exact file counts;
* descriptions likely to become stale after ordinary development.

## 8. Keep the file concise

Use the smallest set of instructions that materially improves agent behavior.

A typical `AGENTS.md` should contain sections such as:

```markdown
# <Repository> agent guide

## Scope and sources of truth

## Repository structure

## Development commands

## Durable constraints

## Testing and validation

## Change guidance
```

Adapt the structure to the repository. Do not create empty or low-value sections merely to follow the template.

Repository structure should describe only major areas an agent needs to navigate.

Development commands should contain commands that are actually useful for modifying or validating the repository.

Durable constraints should receive the most attention.

## 9. Avoid duplicating documentation

Do not reproduce large portions of README files, architecture documents, schemas, or contributor guides.

When existing documentation already explains something well:

* state the operational rule an agent needs;
* point to the authoritative document or file;
* avoid copying its detailed explanation.

`AGENTS.md` should function as a navigation and constraint layer over the repository.

## 10. Account for nested instructions

If the repository is large enough that different subtrees have substantially different workflows or constraints, consider whether nested `AGENTS.md` files would be more appropriate.

Create nested files only when there is a meaningful scope boundary.

Do not fragment instructions merely because the repository contains multiple directories.

Keep repository-wide rules in the root `AGENTS.md`.

## 11. Validate the result

Before finishing:

1. Re-read every instruction in the generated `AGENTS.md`.
2. Verify that each concrete claim is supported by current repository evidence.
3. Remove speculative or generic advice.
4. Remove duplicated README/documentation content.
5. Remove temporary implementation details.
6. Check that documented commands and paths exist.
7. Check that important repository-specific constraints discovered during inspection are represented.
8. Check that the file remains useful if ordinary implementation details change.

If the repository contains contradictory evidence, do not silently choose one interpretation. Either resolve it from stronger sources or describe the relevant source of truth without asserting an unsupported rule.

## 12. Output

Create `AGENTS.md` at the target repository root unless a different scope was explicitly requested.

After creating it, summarize:

* what sources were inspected;
* the major constraints captured;
* any important uncertainties or contradictions discovered;
* whether nested `AGENTS.md` files appear warranted.

Do not modify unrelated repository files.
