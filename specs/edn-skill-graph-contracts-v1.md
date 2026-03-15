---
title: EDN Skill Graph Contracts + Warn-only Lint (v1)
status: refined
owner: err
created_at: 2026-03-15
updated_at: 2026-03-15
tags: [contracts, edn, skill-graph, lint, kanban, opencode, pi]
---

# Summary
Establish a deterministic, lintable, EDN (data-only) contract layer for skills, where:

1. A **skill graph/tree** is a first-order concept represented as EDN list-forms.
2. Skills may **expose followup skill registries** (graph expansion) in their contracts.
3. Global “operation-mindfuck” prompts become **linted EDN** and include a **skill registry** (metadata pointers only).
4. Linting is **warn-only** initially (no blocking), but produces receipts suitable for later reinforcement/gating.
5. Contracts evolve toward **executable** protocol modules with **datalog-like** reasoning over world facts (datoms).

This spec targets canonical skill content in:
- `@devel/orgs/open-hax/opencode-skills/`

And integration points in the local pi/OpenCode runtime:
- `~/.config/opencode/operation-mindfuck/*.lisp`
- `~/.pi/agent/extensions/*` (for warn-only lint + registry awareness)

# Motivation
Current state:
- Kanban/spec-driven-dev workflows exist as many OpenCode skills in `.opencode/skills/…` but are not uniformly compatible with pi’s Agent Skills loader (missing YAML frontmatter in many SKILL.md files; inconsistent naming).
- operation-mindfuck prompts are “Lisp-ish” but not treated as a real language with lint/schema rules.
- The desired direction is **deterministic, reinforceable contracts** with machine-checkable EDN artifacts, eventually executable (rules, gates, receipts).

# Goals
## G1. EDN as canonical contract format
- Contracts are `.edn` files.
- Syntax is **EDN list-forms** (no reader-eval; data-only).

## G2. Skill graph is first-order
- The kernel maintains a base `(skill-registry …)`.
- A loaded skill contract may include `(exposes (skill-registry …))` to expand available skills.
- Registry merge semantics are deterministic.

## G3. Warn-only lint
- All operation-mindfuck prompt files and all `CONTRACT.edn` files are linted by an EDN reader.
- Assistant-emitted EDN blocks (fenced) are linted and warnings recorded.
- No blocking of agent operation in v1.

## G4. Executable direction
- Contracts are structured to compile into a normalized IR.
- Facts/observations can be represented as datoms and reasoned over with datalog rules.

# Non-Goals (v1)
- Hard gating/turn rejection when lint fails.
- Full schema/type system for every form.
- Refactoring all existing kanban skills into generated forms.

# Background / Evidence (local observations)
- `opencode-skills/.opencode/skills` contains 125 skills; 51 missing YAML frontmatter.
- `agile-process/SKILL.md` frontmatter name has a typo (`agile-proces`).
- operation-mindfuck `.lisp` files in `~/.config/opencode/operation-mindfuck/` are currently EDN-readable as list-forms (good baseline for lint).

# Proposed Data Model
## 1) Kernel registry (global)
Stored in operation-mindfuck prompts as metadata pointers only.

```clojure
(skill-system
  (rule "Skills are subordinate modules; they may extend mutable protocol but must not override doctrine/constraints unless user explicitly approves."))

(skill-registry
  (root "~/.agents/skills")
  (entry (name "regression-triage") (contract "~/.agents/skills/regression-triage/CONTRACT.edn") (priority 80))
  (entry (name "fork-tax")          (contract "~/.agents/skills/fork-tax/CONTRACT.edn")          (priority 90))
  (entry (name "spec-driven-dev")   (contract "~/.agents/skills/spec-driven-dev/CONTRACT.edn")   (priority 70)))
```

## 2) Skill contract (node)
Each skill provides a `CONTRACT.edn` (list-forms).

Minimal shape:

```clojure
(skill-contract
  (name "spec-driven-dev")
  (v "ημ.skill/spec-driven-dev@0.1.0")
  (activation
    (priority 70)
    (explicit ["skill:spec-driven-dev"])
    (triggers ["write a spec" "spec draft" "kanban" "phase plan"]))
  (governance
    (touch-layer :mutable)
    (non-override [:mission :directives :safety :license :output-shape])
    (requires-user-approval false))
  (protocol
    (workflow
      ["Create/refresh a spec draft in specs/drafts/"
       "Decompose into phases"
       "Add/maintain todos"
       "Verify after each phase"]))
  (exposes
    (skill-registry
      (entry (name "kanban-curation") (contract "~/.agents/skills/kanban-curation/CONTRACT.edn") (priority 60))
      (entry (name "kanban-fsm")      (contract "~/.agents/skills/kanban-fsm/CONTRACT.edn")      (priority 65))
      (entry (name "task-router")     (contract "~/.agents/skills/task-router/CONTRACT.edn")     (priority 66)))))
```

## 3) Registry expansion semantics
Deterministic algorithm (draft):

- Active registry starts with kernel `(skill-registry …)`.
- When a skill is loaded, add its exposed entries.
- Expand transitively with bounds:
  - `max-depth = 4`
  - `max-nodes = 200`
- Collision resolution (draft):
  1. explicit user invocation wins
  2. higher `(priority N)` wins
  3. if still tied: prefer kernel entry; otherwise warn + require user choice

# Kanban / Spec-driven-dev Cleanup Plan (conceptual)
## Problems to fix
- Many kanban skills lack YAML frontmatter → pi won’t load them as skills.
- State/token naming is inconsistent:
  - `in-progress` vs `in_progress`
  - `review` vs `in_review`
  - 7-state vs extended FSM (incoming/accepted/breakdown/ready/testing/rejected)

## Target end state
- A single source-of-truth **FSM contract** (`kanban-fsm/CONTRACT.edn`) expresses:
  - states
  - allowed transitions
  - per-transition gates
  - WIP caps (optional)
- Existing micro-skills become either:
  - thin wrappers that reference the FSM contract, or
  - generated/documentation-only views

# Linting (Warn-only) Requirements
## R1. Contract file lint (startup)
Lint targets:
- `~/.config/opencode/operation-mindfuck/*.lisp`
- `~/.agents/skills/**/CONTRACT.edn`
- optionally: `@devel/orgs/open-hax/opencode-skills/.opencode/skills/**/CONTRACT.edn` (source repo)

Lint definition:
- EDN reader must parse to EOF without error (enforces (),[],{} correctness).

Output:
- UI warnings summary (counts + top errors)
- Persist receipts (append-only)

## R2. Assistant message EDN lint (warn-only)
Only lint fenced blocks tagged as EDN:
- ```edn
- ```clojure

Rules:
- Each block must be EDN-readable.
- Later: optional schema-check for known forms.

Output:
- UI warning with location (message id / turn index)
- Persist receipt with extracted block hash + parse error

# Receipts / World Facts (direction)
Define a minimal datom-like record for lint results:

```clojure
(obs
  (ctx 己)
  (about (file "~/.config/opencode/operation-mindfuck/ημΠ.dev.v1.lisp"))
  (signal (lint/edn {:ok true :forms 1}))
  (p 1.0)
  (time "2026-03-15T…"))
```

Later, these become inputs to datalog rules (e.g., deny publish/Π if lint violations exist).

# Affected Files / Components (planned)
## Canonical skills repo
- `@devel/orgs/open-hax/opencode-skills/.opencode/skills/**/SKILL.md` (frontmatter normalization)
- new: `@devel/orgs/open-hax/opencode-skills/.opencode/skills/<name>/CONTRACT.edn`
- new: `@devel/orgs/open-hax/opencode-skills/specs/drafts/*` (this spec series)

## Local runtime integration
- `~/.config/opencode/operation-mindfuck/*.lisp` (add kernel skill registry + governance; keep EDN-parseable)
- new pi extension: `~/.pi/agent/extensions/opmf-edn-lint.ts` (warn-only)
- optional: `~/.pi/agent/extensions/opmf-skill-graph.ts` (registry expansion awareness)

# Open Questions
1. Canonical Kanban state tokens:
   - prefer hyphenated (`in-progress`) or underscored (`in_progress`)?
   - requirement: skill names must remain kebab-case for Agent Skills compatibility.
2. Where should `CONTRACT.edn` live for skills distributed via `.opencode/skills`?
   - colocated in each skill directory (recommended)
3. Should `agile-process` be split into:
   - `kanban-fsm` (data)
   - `kanban-playbook` (human guidance)
4. What is the authoritative FSM document:
   - `docs/reference/process.md` (mentioned by validators) exists in which repo(s)?
5. Receipt storage location for lint warnings:
   - global (`~/.config/opencode/lint/…`) vs per-repo (`.ημ/…`)

# Risks
- Contract/schema creep: over-modeling too early.
- Naming migration churn across many micro-skills.
- Duplicated skill corpora (source repo vs symlinked installs) causing drift.

# Priorities
P0: Warn-only EDN lint for global mindfuck prompts.
P0: Introduce `CONTRACT.edn` for the three root skills (regression-triage, fork-tax, spec-driven-dev).
P1: Define `kanban-fsm` contract as the single source-of-truth.
P1: Normalize frontmatter for kanban/spec skills so pi can load them.
P2: Start capturing lint receipts as datoms for later datalog gating.

# Implementation Phases (draft)
## Phase 1 — Spec + scaffolding
- Add `specs/drafts/*` documents
- Define contract schemas and registry semantics

## Phase 2 — Warn-only lint extension
- Implement `opmf-edn-lint` extension (startup + message_end)
- Record lint receipts

## Phase 3 — Root contracts + registry
- Add `CONTRACT.edn` for root skills
- Add `(skill-registry …)` to operation-mindfuck kernel

## Phase 4 — Kanban cleanup
- Add frontmatter to missing kanban/spec skills
- Fix naming mismatches (e.g., agile-proces typo)
- Introduce `kanban-fsm/CONTRACT.edn`

# Definition of Done (v1)
- A documented EDN contract schema exists (this spec + followups).
- Warn-only lint runs and produces receipts for:
  - operation-mindfuck prompt files
  - `CONTRACT.edn` files
  - assistant-emitted fenced EDN blocks
- A root skill registry exists (kernel) and points to three root contracts.
- A `spec-driven-dev` contract exposes followup registry entries for kanban management.
