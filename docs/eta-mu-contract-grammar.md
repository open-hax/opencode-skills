# ημ Contract Grammar (Plain English)

This document describes the *grammar* and *conventions* behind the ημ (“eta-mu”) methodology as it’s currently practiced in this repo + the associated `operation-mindfuck` kernel.

It’s written for teams who want to **adopt the methodology** without needing to read all the source prompt files.

> Scope: This is a living system. Some parts are already enforced (e.g. EDN linting, skill discovery), others are “directional” (eventually executable contracts, datalog reasoning).

---

## 1) What this is

We treat agent behavior and workflow as a **contract system** described in **EDN list-forms** (S-expressions):

- **Kernel contract**: stable doctrine + constraints + registries.
- **Skill contracts**: modular procedures that can be loaded on-demand.
- **Facts/observations/rules**: a datalog-like path for reasoning over “world state” with explicit provenance.

### Why EDN list-forms?

- Deterministic parsing (balanced `()[]{};` strings/comments).
- Portable across editors/tooling (no bespoke syntax).
- Simple path from “documentation” → “linted contract” → “executable interpreter”.

**Rule of thumb:** if it’s meant to be checked or executed later, it should be expressible as EDN data.

---

## 2) The symbolic operators you can use (modes)

The assistant recognizes *mode operators* (single-token switches). These change how responses are shaped.

- `η` — **Delivery mode**: minimal executable core; no hedges; no questions unless blocked.
- `μ` — **Formal mode**: smallest adequate formalism (types/spec/math); crisp definitions.
- `Π` — **Fork Tax mode**: deterministic handoff snapshot (commit/tag/push + manifests).
- `A` — **Art mode**: creative output allowed, but still precise about constraints/claims.

**Precedence:** if multiple appear, use: `Π > μ > η > A`.

---

## 3) Context glyphs (where claims come from)

When emitting structured facts/observations, we use context symbols to attribute claims:

- `己` — the agent (“self”) / internal reasoning
- `汝` — the user
- `彼` — third parties (“them”)
- `世` — the external world (things that require evidence)
- `主` — presence/attention anchor (marker, not evidence)

**Method rule:** do not present world claims (`世`) as facts without provenance/evidence.

---

## 4) Default response shape

Unless a mode changes it (especially `η`), responses are structured into these sections:

- **Signal** — the actual deliverable
- **Evidence** — citations/tool refs used *this turn*
- **Frames** — 2–3 plausible interpretations
- **Countermoves** — checks against misinterpretation/manipulation
- **Next** — exactly one small next action

This creates predictable outputs and supports later “receipts” / audit trails.

---

## 5) EDN list-form style

### 5.1 General style

An EDN list-form looks like:

```clojure
(form-name
  (field-a "value")
  (field-b 123)
  (field-c ["x" "y"]))
```

Conventions:
- The first symbol is the **form name** (`form-name`).
- Each subsequent item is usually a **clause** like `(field value...)`.
- Use strings for human text, keywords for stable IDs.
- Avoid evaluation features; treat EDN as **data only**.

### 5.2 “MUST be lintable”

All contract files are intended to be **EDN-reader parseable**. This implies:
- balanced `()[]{}'
- valid strings
- valid comment usage

In your runtime, EDN linting is currently **warn-only**.

---

## 6) Skills: contracts + registries + graph expansion

### 6.1 Skill discovery (human layer)

A skill is primarily discovered via `SKILL.md` (Agent Skills format):

```yaml
---
name: my-skill
description: "What it does, when to use it"
---
```

### 6.2 Skill contract (machine layer)

Every skill may also include a `CONTRACT.edn` alongside `SKILL.md`.

A minimal skill contract form:

```clojure
(skill-contract
  (name "kanban-fsm")
  (v "ημ.skill/kanban-fsm@0.1.0")
  (intent "Define the canonical Kanban FSM as data")
  (activation (priority 65) (triggers ["status transition"]))
  (governance (touch-layer :mutable))
  (effects (writes false) (network false) (commits false))
  (exposes
    (skill-registry
      (entry (name "child-skill") (contract "~/.agents/skills/child-skill/CONTRACT.edn") (priority 60)))))
```

Important fields (plain meaning):
- `(name ...)` — stable identifier
- `(v ...)` — version tag
- `(activation ...)` — how this skill is selected (explicit commands, triggers, priority)
- `(governance ...)` — what the skill is allowed to touch (prevents overriding doctrine)
- `(effects ...)` — what the skill may do (write files, network, commits)
- `(exposes ...)` — optional: adds followup skills to the active registry

### 6.3 Skill registry (the graph)

A registry is a list of entries:

```clojure
(skill-registry
  (root "~/.agents/skills")
  (entry (name "regression-triage") (contract ".../CONTRACT.edn") (priority 80))
  ...)
```

**Skill graph rule:** skills may expose registries (children). The active registry is:

> kernel registry ∪ registries exposed by loaded skills (bounded, deterministic)

If multiple skills match, selection should be deterministic (priority or explicit choice).

---

## 7) Kanban FSM contracts (data, not prose)

The Kanban workflow is modeled as an explicit FSM contract, e.g.:

```clojure
(kanban-fsm
  (normalization ...)
  (invariants ...)
  (states ...)
  (transitions ...)
  (gates ...))
```

Plain English meanings:
- **states**: what statuses exist and what they mean
- **transitions**: allowed moves (from → to)
- **gates**: checks required to take a move
- **invariants**: rules that must hold for a task to be valid in a state
- **normalization**: legacy aliases (e.g. `in_progress` → `in-progress`)

Method rule (normative): when taking a transition, validate:
1) edge exists
2) listed gates pass
3) destination-state invariants hold

---

## 8) Facts, observations, questions, and rules (datalog direction)

We use a small set of “knowledge forms” designed for later datalog-style reasoning:

- `fact` — a claim treated as true within a scope, with provenance and confidence
- `obs` — an observation/signal (may be uncertain)
- `q` — an open question (what is missing / why blocked)
- `rule` — a derivation rule (future: datalog)

Example:

```clojure
(fact (ctx 世) (claim "X is true") (source "url-or-tool") (p 0.9) (time "...") )
(obs  (ctx 己) (about "file") (signal "lint ok") (p 1.0))
(q    (ctx 汝) (ask "What is the repo root?") (why-blocked "needed for git ops"))
```

**Key discipline:** separate:
- what was observed (`obs`)
- what is asserted as world truth (`fact`) with evidence
- what is unknown (`q`)

---

## 9) Emitting EDN in assistant messages

If the assistant emits contract/fact forms in chat, they should be placed in fenced blocks:

```edn
(fact ...)
```

This enables runtime linting (warn-only today; enforceable later).

---

## 10) Adoption checklist (minimal)

1. **Adopt EDN list-forms** for any workflow you want lintable/executable.
2. Keep kernel doctrine/constraints stable; move procedures into skills.
3. For each skill:
   - keep `SKILL.md` short + clear
   - add `CONTRACT.edn` for machine-checked behavior
4. Model complex workflows as **data contracts** (e.g., Kanban FSM), not dozens of duplicated procedural docs.
5. Emit EDN forms only inside fenced blocks.

---

## Appendix: Common pitfalls

- YAML frontmatter `description:` must be quoted if it contains `:`.
- Avoid recursive symlinks inside skill directories (can cause infinite discovery).
- Don’t rely on “board ordering” as an enforcement mechanism; encode what is policy vs gate.

