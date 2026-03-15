---
title: Kanban FSM Contract (EDN, list-forms) v1
status: refined
owner: err
created_at: 2026-03-15
updated_at: 2026-03-15
tags: [kanban, fsm, contracts, edn, datalog, skill-graph]
---

# Summary
Define a single, canonical **Kanban finite state machine** (FSM) as a **data-only EDN contract** (list-forms) that can be:

- loaded on-demand as a skill contract (`kanban-fsm/CONTRACT.edn`)
- linted by an EDN reader (balanced `()[]{};` strings/comments)
- compiled later into deterministic validators
- used as the source-of-truth for existing micro-skills:
  - `validate-*-to-*`
  - `work-on-*-task`
  - `kanban-state-*`

This contract replaces the currently referenced-but-missing `docs/reference/process.md` as the authoritative transition table.

# Motivation
In `opencode-skills/.opencode/skills`, Kanban workflow is spread across many skills and references an external doc:

- Many skills claim: “Validates … using `docs/reference/process.md` rules”
- `docs/reference/process.md` does not exist in this repo
- Two vocabularies exist simultaneously:
  - **Simple 7-state** board (legacy): `icebox backlog todo in-progress in-review document done`
  - **Extended** workflow used by validators/routers: `icebox incoming accepted breakdown blocked ready todo in-progress in-review testing document done rejected`

A canonical FSM contract makes the workflow deterministic, lintable, and eventually executable.

# Goals
1. Express the FSM as **pure EDN data** using **list-forms**.
2. Provide **canonical internal state identifiers** plus **surface token aliases** (to absorb `in_progress` vs `in-progress`, `in_review` vs `review`, etc.).
3. Provide a deterministic **transition table** with per-transition gate references.
4. Provide a place to define **WIP limits** as data (optional).
5. Be friendly to future **datalog-like reasoning** (rules can be added without changing the core schema).

# Non-Goals (v1)
- Automatically generating/updating all micro-skill markdown.
- Enforcing gates at runtime (that will come after warn-only lint + receipts).

# Canonical Concepts
## Task
A “task” is a file (typically a spec) with YAML frontmatter including a workflow status.

## Priority
### Priority scale
Tasks in `todo` MUST have an integer `priority` in frontmatter.

- Range: `0..5`
- Semantics:
  - `0` = urgent (treat as priority **1** for ordering, but flag as urgent)
    - REQUIRED (todo only): frontmatter `urgency-reason` (string)
  - `1..4` = normal priority scale (1 highest, 4 lowest)
  - `5` = deferred/low priority; tasks at priority 5 SHOULD be deferred to `icebox` (not kept in `todo`)

### Ready defaults
- `ready` tasks MAY omit `priority`.
- If `priority` is missing in `ready`, there is **no priority contract** set; it is inferred to be **low importance**.
  - This is a conceptual inference only.
  - Normalizers MUST NOT materialize/write `priority: 5` into frontmatter.
  - Missing `priority` is NOT considered “explicit priority” for the `ready → todo` commitment gate.

### Selection policy
- Board ordering is a suggestion; workers may pick a lower-priority item if better suited.

## State
A state has:
- a canonical internal `id` (keyword)
- one preferred frontmatter token string
- zero or more alias tokens

## Transition
A transition is a directed edge `from → to`, optionally with:
- gate checks (by ID)
- a suggested micro-skill name (legacy compatibility)

Validation semantics (normative): when taking a transition, the system MUST evaluate:
1. edge exists in `(transitions ...)`
2. all listed gates for that transition
3. all destination-state invariants (i.e., invariants that apply to the resulting state)

## Gate
A gate is a named check that must pass to take a transition.

In v1 we keep gates declarative and **implementation-agnostic**:
- a gate has an `id`
- a human description
- optionally a *checklist* and/or a *rule stub* (for future datalog)

# Contract File Location
Recommended canonical location in this repo:

- `.opencode/skills/kanban-fsm/CONTRACT.edn`

Installation paths (later):
- `~/.agents/skills/kanban-fsm/CONTRACT.edn`

# Contract Shape (EDN list-forms)
The contract file SHOULD be readable as a sequence of EDN forms.

## Top-level form
```clojure
(kanban-fsm
  (v "ημ.kanban/fsm@0.1.0")
  (source "opencode-skills")
  (profiles ...)
  (states ...)
  (transitions ...)
  (gates ...)
  (wip ...)
  (invariants ...)
  (normalization ...))
```

### (profiles …)
Profiles allow coexisting “views” without duplicating the FSM.

Example:
```clojure
(profiles
  (profile
    (id :profile/extended)
    (description "incoming→accepted→breakdown→ready→todo→in-progress→in-review→testing→document→done")
    (primary true))
  (profile
    (id :profile/simple-7)
    (description "icebox→backlog→todo→in-progress→in-review→document→done")
    (primary false)
    (projection
      ;; Projection rules map canonical states to view states.
      ;; (Details TBD; v1 can keep this as prose.)
      (notes "Normalize tokens: backlog→ready, triage→breakdown, review→in-review"))))
```

### (states …)
Each state is declared once.

```clojure
(states
  (state
    (id :kanban.state/icebox)
    (frontmatter (preferred "icebox") (accept ["icebox"]))
    (label "Icebox")
    (kind :intake)
    (description "Unprioritized ideas; may be rejected or promoted."))

  (state
    (id :kanban.state/in-progress)
    (frontmatter (preferred "in-progress") (accept ["in-progress" "in_progress"]))
    (label "In Progress")
    (kind :execution)
    (description "Implementation in flight."))

  ;; Canonical internal state id decision: :kanban.state/in-review
  (state
    (id :kanban.state/in-review)
    (frontmatter (preferred "in-review") (accept ["in-review" "in_review" "review"]))
    (label "In Review")
    (kind :qa)
    (description "PR review / QA gate.")))
```

Notes:
- Use a **keyword** `id` as the canonical key for datalog + deterministic reasoning.
- Keep frontmatter tokens as strings to match real YAML values.

### (transitions …)
Transitions reference state IDs.

```clojure
(transitions
  (transition
    (id :kanban.transition/icebox->incoming)
    (from :kanban.state/icebox)
    (to :kanban.state/incoming)
    (gates [])
    (legacy-skill "validate-icebox-to-incoming"))

  (transition
    (id :kanban.transition/breakdown->ready)
    (from :kanban.state/breakdown)
    (to :kanban.state/ready)
    (gates [:kanban.gate/ready-storypoints :kanban.gate/ready-acceptance-criteria])
    (legacy-skill "validate-breakdown-to-ready"))

  (transition
    (id :kanban.transition/ready->todo)
    (from :kanban.state/ready)
    (to :kanban.state/todo)
    (gates [:kanban.gate/todo-priority-present :kanban.gate/todo-priority-range])
    ;; Note: urgency-reason when priority==0 is enforced by destination-state invariants, not listed here.
    ;; Note: if priority is missing, this transition MUST be rejected and the task remains in ready.
    (legacy-skill "validate-ready-to-todo"))

  (transition
    (id :kanban.transition/ready->icebox)
    (from :kanban.state/ready)
    (to :kanban.state/icebox)
    (gates [])
    ;; Defer: triaged but not worth scheduling/committing.
    ;; Common case: priority==5 implies the task MUST be in icebox.
    (legacy-skill "validate-ready-to-icebox"))

  (transition
    (id :kanban.transition/todo->icebox)
    (from :kanban.state/todo)
    (to :kanban.state/icebox)
    (gates [])
    ;; Decommit/defer: used when a committed item is no longer worth scheduling.
    ;; Common case: priority==5 implies the task MUST be in icebox.
    (legacy-skill "validate-todo-to-icebox")))
```

### (gates …)
Gates are named checks.

```clojure
(gates
  (gate
    (id :kanban.gate/ready-storypoints)
    (description "Task must be sized (storyPoints) and <= 5")
    (checklist
      ["frontmatter.storyPoints exists"
       "frontmatter.storyPoints <= 5"]))

  (gate
    (id :kanban.gate/ready-acceptance-criteria)
    (description "Acceptance criteria checklist exists")
    (checklist
      ["task body contains an Acceptance Criteria checklist"]))

  (gate
    (id :kanban.gate/todo-priority-present)
    (description "Todo requires an explicit frontmatter priority")
    (checklist
      ["frontmatter.priority exists"
       "if missing: reject transition to todo and keep task in ready (bounce back)"]))

  (gate
    (id :kanban.gate/todo-priority-range)
    (description "priority is an integer in range 0..5; todo allows 0..4 (5 means defer to icebox)")
    (checklist
      ["frontmatter.priority is integer"
       "0 <= frontmatter.priority <= 5"
       "if state==todo then frontmatter.priority <= 4"
       "if frontmatter.priority == 5 then transition to icebox (ready->icebox or todo->icebox depending on current state)"]))

  (gate
    (id :kanban.gate/todo-urgency-reason-if-zero)
    (description "If priority==0, frontmatter.urgency-reason must exist and be non-empty")
    (checklist
      ["if frontmatter.priority == 0 then frontmatter.urgency-reason exists"
       "frontmatter.urgency-reason is a non-empty string"]))

  (gate
    (id :kanban.gate/review-artifact)
    (description "A reviewable artifact exists (PR link or commit reference)")
    (checklist
      ["task body contains PR/commit link"
       "CI is green (if applicable)"]))

  (gate
    (id :kanban.gate/tests-green)
    (description "Tests must pass")
    (checklist
      ["unit/integration tests pass"
       "lint/typecheck pass"]))

  (gate
    (id :kanban.gate/dependency-real)
    (description "Blocked status requires a real dependency and bidirectional linking")
    (checklist
      ["dependency described"
       "reverse link present"])) )
```

Later evolution:
- gates MAY embed a datalog rule stub:
  - `(rule (id ...) (when ...) (then ...))`
  - or a `q` form
…but v1 keeps this optional.

### (wip …)
Optional limits.

```clojure
(wip
  (cap (state :kanban.state/in-progress) (limit 3) (scope :repo))
  (cap (state :kanban.state/in-review)   (limit 5) (scope :repo)))
```

### (invariants …)
Global invariants.

```clojure
(invariants
  (must "Every task has exactly one normalized state")
  (must "State token normalizes to a declared :kanban.state/*")
  (must "Transitions only occur along declared edges")
  (must "Tasks in todo have frontmatter.priority (int 0..4; 0 marks urgency)")
  (must "If priority==5 then task MUST be in icebox (deferred)")
  (must "If state==todo and priority==0 then frontmatter.urgency-reason is required")
  (must "tags is canonical; labels is legacy and must be normalized"))
```

### (normalization …)
Define token normalization and legacy field rules.

```clojure
(normalization
  (status-field
    (canonical "status")
    (accept ["status" "state"]))
  (tags-field
    (canonical "tags")
    (legacy "labels"))
  (token-aliases
    ;; canonicalize legacy underscore (and other) tokens to kebab-case
    (alias "in_progress" "in-progress")
    (alias "in_review"   "in-review")
    (alias "review"      "in-review")
    (alias "backlog"     "ready")
    (alias "triage"      "breakdown"))
  (frontmatter-aliases
    ;; canonicalize legacy underscore keys to kebab-case
    (alias "urgency_reason" "urgency-reason")))
```

# Canonical State Set (proposal)
Proposed canonical states (extended profile):

- `icebox`
- `incoming`
- `accepted`
- `breakdown`
- `blocked`
- `ready`
- `todo`
- `in-progress` (alias: `in_progress`)
- `in-review` (aliases: `in_review`, `review`)
- `testing`
- `document`
- `done`
- `rejected`

Notes:
- `backlog` is an alias token for `ready`.
- `triage` is an alias token for `breakdown`.

# Compatibility with Existing Micro-Skills
## Observed mismatch
Many `validate-*` skills currently include a copy/pasted “transition-specific gates” list that does not actually specialize per transition.

The FSM contract becomes the source-of-truth for:
- allowed transitions
- which gates apply to which transitions

Micro-skill markdown can be simplified to:
- “validate X→Y by consulting kanban-fsm contract and evaluating gates”

# Open Questions
1. Should we include `document → done` as mandatory, or allow `in-review → done` as rare fast-path?

# Risks
- Overfitting to one repo’s process vocabulary.
- Introducing yet another source of truth if we don’t deprecate `docs/reference/process.md` references.

# Definition of Done (for this spec)
- A concrete `kanban-fsm` EDN contract schema exists.
- The canonical state set + alias strategy is explicitly documented.
- Transition + gate representation is explicit enough to implement an interpreter later.
