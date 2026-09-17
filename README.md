# Affitio Agentic Development Operating Model
## Project Lead → Orchestrator → Worker Pattern
### Repository Study, Design, and Implementation Specification

**Status:** Proposed architecture for investigation and implementation  
**Project:** Affitio  
**Primary development environment:** Cursor + Cursor Cloud Agents  
**Purpose:** Give a coding agent with direct repository access enough context, constraints, questions, and expected outputs to study how this multi-agent development model should be applied to the real Affitio codebase.

---

# 1. Purpose

Affitio intends to adopt a structured multi-agent software development operating model built around three primary execution roles:

1. **Project Lead**
2. **Implementation Orchestrator**
3. **Worker**

A fourth supporting role, **Verifier**, is recommended for independent review.

The goal is not simply to run multiple coding agents in parallel. The goal is to create a development system where:

- agents can safely work in parallel;
- new agents can start with little or no conversational context;
- work can survive context-window exhaustion;
- responsibilities and ownership are explicit;
- project state is persisted in the repository;
- architecture and product decisions do not disappear with an agent session;
- implementation agents cannot silently expand scope;
- parallel agents do not corrupt shared project state;
- a fresh agent can resume another agent's work from repository artifacts;
- work is independently verified before being considered complete;
- the user can primarily interact with a capable Project Lead rather than micromanaging many workers;
- agent output remains understandable and auditable by a human.

The central principle is:

> **Agents are replaceable compute. The repository is the persistent project brain.**

The desired system should use Git, repository documents, Cursor Rules, Skills, Agents/Subagents, hooks, branches/worktrees/Cloud Agents, and CI as appropriate.

This document describes the desired operating model. The coding agent receiving this document must inspect the actual Affitio repository and determine the safest and simplest way to implement it in practice.

---

# 2. Important instruction to the coding agent

Do **not** immediately implement everything described here.

This document intentionally includes ideas at different levels of maturity.

Your first responsibility is to:

1. inspect the actual repository;
2. understand the existing monorepo structure;
3. inspect existing Cursor configuration;
4. inspect existing project documentation;
5. inspect existing CI/CD and testing;
6. inspect current Git/branching assumptions;
7. identify conflicts with the proposed operating model;
8. verify what current Cursor capabilities are actually available;
9. determine what should be implemented now versus later;
10. propose a concrete Affitio-specific implementation plan.

Treat this document as a **target operating model and design hypothesis**, not as permission to blindly create files.

Where this specification conflicts with an existing, intentional Affitio architecture decision, surface the conflict explicitly.

Do not silently replace existing project conventions.

---

# 3. Background and motivation

Affitio will make heavy use of Cursor Cloud Agents.

Important characteristics of this environment include:

- an agent may begin with little or no prior conversational context;
- different agents may run in separate environments;
- agents may run for long periods;
- context windows will eventually be compacted or exhausted;
- multiple agents may work simultaneously;
- individual agents should therefore not be treated as permanent holders of project knowledge.

A conversational history alone is not a sufficient project-management system.

The proposed solution is to persist the minimum useful operational state inside the repository and separate it into clear ownership scopes.

---

# 4. Core conceptual model

The default authority hierarchy is:

```text
                                  USER
                                    │
                                    │
                          product/business authority
                                    │
                                    ▼
                         ┌────────────────────┐
                         │    PROJECT LEAD    │
                         │ global control     │
                         │ plane              │
                         └─────────┬──────────┘
                                   │
                  ┌────────────────┼────────────────┐
                  │                                 │
          small bounded task                  large milestone
                  │                                 │
                  ▼                                 ▼
             ┌────────┐                   ┌──────────────────┐
             │ Worker │                   │   ORCHESTRATOR   │
             └────────┘                   └────────┬─────────┘
                                                  │
                                      ┌───────────┼───────────┐
                                      ▼           ▼           ▼
                                   Worker      Worker      Worker
                                      │           │           │
                                      └───────────┼───────────┘
                                                  ▼
                                              Verifier
                                                  │
                                                  ▼
                                           Orchestrator
                                              review
                                                  │
                                                  ▼
                                            Project Lead
                                             final gate
```

The normal management depth should remain shallow.

Recommended maximum project-level delegation depth:

```text
Project Lead
    ↓
Orchestrator
    ↓
Worker
```

Avoid uncontrolled recursive hierarchies such as:

```text
Project Lead
  → Orchestrator
    → Sub-Orchestrator
      → Worker Manager
        → Worker
```

unless there is a demonstrated future need.

---

# 5. Authority model

The user remains the ultimate product/business authority.

Suggested responsibility boundaries:

| Decision | Default authority |
|---|---|
| Business model | User |
| Product direction | User |
| Product priority | User + Project Lead |
| Major product scope | User + Project Lead |
| Major architecture | Project Lead recommends; user approves where material |
| Cross-domain architecture | Project Lead |
| Large task decomposition | Orchestrator |
| Local implementation details | Worker |
| Task acceptance | Orchestrator |
| Milestone acceptance | Project Lead |
| Business acceptance | User |
| Scope expansion | Parent approval required |
| Durable architecture decision | Project Lead promotes to canonical documentation |

The system should minimize unnecessary escalation.

Workers should not ask the user about normal implementation details.

Orchestrators should not ask the user about decisions their task contract already delegates to them.

Project Leads should involve the user when a decision materially changes product direction, business behavior, cost, risk, architecture, or committed scope.

---

# 6. Role 1 — Project Lead

## 6.1 Purpose

The Project Lead is the primary agent the user interacts with.

It acts as the global control plane for the Affitio development effort.

Its job is not merely task tracking.

It should understand:

- what Affitio is;
- current product goals;
- major architecture;
- important technical constraints;
- business decisions;
- active milestones;
- dependencies;
- risks;
- work currently delegated;
- project state;
- important unresolved questions.

The Project Lead should be capable enough to challenge unclear requirements and identify missing decisions before implementation begins.

## 6.2 Responsibilities

The Project Lead should:

- discuss product ideas with the user;
- challenge assumptions;
- identify important missing requirements;
- identify technical implications;
- identify business implications;
- distinguish decisions that require the user from decisions that can be delegated;
- convert ideas into implementation-ready task briefs;
- maintain the project roadmap;
- maintain the active project state;
- maintain cross-task dependency awareness;
- delegate bounded work;
- decide whether a task needs an Orchestrator;
- decide whether a task can go directly to a Worker;
- review Orchestrator output;
- review relevant evidence;
- identify missing work;
- request iteration when quality is insufficient;
- promote durable decisions from task-level documents into canonical project documentation;
- keep project-level context current enough that a replacement Project Lead can resume work.

## 6.3 Project Lead should normally not write application code

The Project Lead may inspect code, search code, read diffs, run tests, inspect CI/PRs, update project-control documents, create task briefs, and review implementation.

It should normally **not** directly edit production/application code.

If it discovers that code changes are required, it should create or delegate a Worker task.

The coding agent should assess whether limited exceptions are necessary in practice.

---

# 7. Role 2 — Implementation Orchestrator

## 7.1 Purpose

An Orchestrator owns **one bounded large task, phase, or milestone**.

Examples:

```text
O-021 Creator Search V1
O-022 Usage Accounting
O-023 YouTube Ingestion Reliability
O-024 Contact Discovery MVP
O-025 Stripe Entitlement Integration
```

An Orchestrator does not own the entire Affitio project.

It owns the successful delivery of its task contract.

## 7.2 Responsibilities

The Orchestrator should:

- read the Project Lead task brief;
- inspect relevant code and documentation;
- identify missing details;
- ask necessary clarifying questions;
- avoid asking questions already answered in the task brief;
- decompose the task;
- identify shared contracts;
- identify dependencies;
- determine what can run in parallel;
- determine what must be sequential;
- assign bounded Worker tasks;
- assign explicit file/module ownership where practical;
- monitor Worker results;
- integrate results;
- review Worker claims;
- invoke independent verification;
- request fixes;
- keep task STATUS current;
- keep HANDOFF current;
- document task-local decisions;
- report final result upward to Project Lead.

## 7.3 Orchestrator should normally not implement application code

The default model is:

```text
Orchestrator plans and manages
Workers implement
Verifier verifies
```

The Orchestrator may inspect and run code.

It should generally avoid becoming another Worker.

---

# 8. Role 3 — Worker

## 8.1 Purpose

A Worker performs one narrow piece of implementation or investigation.

Workers are intended to have much less context than Project Leads or Orchestrators.

A Worker should not need the entire project history.

It should receive exactly enough context to complete its assignment safely.

## 8.2 Worker types

Possible profiles:

- Implementation Worker
- Investigation Worker
- UI Worker
- Migration Worker
- Documentation Worker
- Fix Worker

These may be separate Cursor agents or reusable Skills depending on what best fits the repository.

## 8.3 Worker responsibilities

Each Worker should:

- read its assignment;
- inspect only the necessary supporting context;
- stay within scope;
- avoid modifying unrelated areas;
- follow explicit ownership boundaries;
- write tests where appropriate;
- run required validation;
- document what changed;
- document deviations;
- document remaining risks;
- produce evidence;
- report back to the parent.

Workers should not independently expand project scope.

Workers should not modify global project-management state.

Workers should not make major product or architecture decisions without escalation.

---

# 9. Supporting role — Verifier

The Verifier exists to create independent evidence.

The implementation agent should not be the final judge of its own output.

Recommended pattern:

```text
Worker
  ↓
"I completed the implementation."
  ↓
Verifier
  ↓
"Prove that the acceptance criteria are actually satisfied."
```

The Verifier should ideally start with fresh context.

It should receive:

- task acceptance criteria;
- relevant implementation result;
- necessary code references;
- verification commands.

The Verifier should not simply trust the Worker's RESULT.

It should inspect code, run tests, inspect runtime behavior where applicable, check edge cases, check acceptance criteria, identify regressions, and report discrepancies.

Preferably the Verifier should be read-only.

If fixes are necessary:

```text
Verifier
   ↓
Orchestrator
   ↓
Fix Worker
   ↓
Verifier
```

---

# 10. Execution modes

The Project Lead should classify tasks before delegation.

## DIRECT

Project Lead handles planning/research itself. No application-code changes.

## SINGLE_WORKER

```text
Project Lead
   ↓
Worker
```

Use for isolated, low-risk, contained work.

## PARALLEL

```text
Project Lead
   ↓
Orchestrator
   ├─ Worker A
   ├─ Worker B
   └─ Worker C
```

Use when slices are sufficiently independent.

## PHASED

Use when dependencies exist.

```text
Phase 1
Shared foundation / contract
      ↓
Phase 2
┌────────────┬────────────┬────────────┐
Backend      Frontend      Tests
└────────────┴────────────┴────────────┘
      ↓
Phase 3
Integration
      ↓
Phase 4
Independent verification
```

---

# 11. Parallelism principle

> **Parallelize implementation only after shared contracts are sufficiently stable.**

Bad:

```text
Worker A changes schema
Worker B builds API against previous schema
Worker C builds UI against a guessed API
```

Better:

```text
Foundation task
  ↓
stabilize:
- types
- schemas
- interfaces
- API contract
  ↓
parallel workers
```

---

# 12. Parallel Orchestrators

Parallel Orchestrators are encouraged.

```text
                         Project Lead
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
 O-021 Creator Search   O-022 Billing       O-023 Ingestion
        │                     │                     │
      Workers                Workers               Workers
```

Each Orchestrator must have a bounded task, unique task ID, explicit scope, clear ownership, isolated operational documents, a branch/worktree/Cloud Agent strategy, known shared dependencies, and an escalation mechanism.

---

# 13. Cross-Orchestrator dependencies

Parallel Orchestrators must not silently modify one another's domains.

Example:

```text
O-021 Creator Search
needs CreatorSchema v3

O-023 Ingestion
owns CreatorSchema v3 changes
```

Recommended flow:

```text
O-021
  ↓
dependency request
  ↓
Project Lead
  ↓
coordinate with O-023
```

The Project Lead owns the global dependency graph.

---

# 14. Parallel Project Leads

Multiple global Project Leads create a split-brain risk.

Recommended default:

> **One active global Project Lead has write authority over global project state.**

Lead-level advisory agents can still run in parallel, e.g.:

- Architecture Reviewer
- Security Reviewer
- Product Strategist
- Cost Reviewer
- Database Reviewer
- Reliability Reviewer
- Agent Architecture Reviewer

These should normally be read-only/advisory.

---

# 15. Future domain Project Leads

If Affitio becomes much larger, authority may be partitioned:

```text
                            USER
                              │
                         Program Lead
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     Discovery Lead    Campaign Lead    Platform Lead
```

This should not be introduced prematurely.

---

# 16. Persistent repository memory

Do not rely on one enormous project-memory file.

Separate:

```text
Durable project memory
+
Task-local operational memory
```

---

# 17. Proposed repository control structure

The coding agent should evaluate whether this fits the actual repository.

```text
Affitio/
│
├── AGENTS.md
├── docs/
│   ├── architecture/
│   ├── product/
│   └── adr/
│
├── .agents/
│   ├── project/
│   │   ├── PROJECT.md
│   │   ├── ROADMAP.md
│   │   ├── STATE.md
│   │   └── DECISIONS.md
│   │
│   ├── runs/
│   │   ├── O-021-creator-search/
│   │   │   ├── BRIEF.md
│   │   │   ├── PLAN.md
│   │   │   ├── STATUS.md
│   │   │   ├── DECISIONS.md
│   │   │   ├── HANDOFF.md
│   │   │   └── workers/
│   │   │       ├── W-021-01/
│   │   │       │   ├── ASSIGNMENT.md
│   │   │       │   └── RESULT.md
│   │   │       └── W-021-02/
│   │   │           ├── ASSIGNMENT.md
│   │   │           └── RESULT.md
│   │   └── O-022-billing/
│   │
│   ├── archive/
│   └── runtime/
│       ├── sessions/
│       ├── locks/
│       └── checkpoints/
│
└── .cursor/
    ├── rules/
    ├── agents/
    ├── skills/
    ├── hooks/
    └── hooks.json
```

---

# 18. Project-level files

## PROJECT.md

Slow-changing orientation:

- mission;
- product principles;
- current scope;
- non-goals;
- architecture map;
- canonical documentation links;
- major constraints.

Avoid duplicating entire architecture documents.

## ROADMAP.md

Milestone-level visibility only.

## STATE.md

Current project dashboard:

- active work;
- blocked work;
- dependencies;
- pending user decisions;
- project risks;
- recent completion;
- likely next work.

## DECISIONS.md

Concise durable decision index.

Large architecture decisions should still use ADRs if appropriate.

---

# 19. Task-level Orchestrator workspace

Each significant task gets an isolated workspace:

```text
.agents/runs/O-021-creator-search/
```

---

# 20. BRIEF.md

Owned by Project Lead.

Suggested sections:

```markdown
# Objective
# Why this matters
# Business outcome
# In scope
# Out of scope
# User-visible behavior
# Acceptance criteria
# Relevant architecture
# Constraints
# Known decisions
# Open questions
# Dependencies
# Required verification
# Deliverables
# Escalation rules
```

---

# 21. PLAN.md

Owned by Orchestrator.

Suggested sections:

```markdown
# Current understanding
# Technical approach
# Dependency graph
# Shared contracts
# Phases
# Worker assignments
# File/module ownership
# Risks
# Rollback/recovery considerations
```

---

# 22. STATUS.md

Owned by Orchestrator.

Workers should not update it directly.

Example:

```text
Overall: ACTIVE

W-021-01 Backend
Status: COMPLETE

W-021-02 UI
Status: ACTIVE

W-021-03 Tests
Status: BLOCKED

Integration:
NOT STARTED

Risks:
- query latency not benchmarked

Next:
- finish UI
- unblock tests
```

---

# 23. Task DECISIONS.md

Contains local decisions.

Project-wide decisions should be escalated and promoted by Project Lead.

---

# 24. HANDOFF.md

Designed for replacing an agent session.

Suggested sections:

```markdown
# Task goal
# Current state
# What is complete
# What is in progress
# What is blocked
# Important decisions
# Important discoveries
# Current branches / commits / PRs
# Verification already performed
# Known failures
# Risks
# Exact next actions
# Files the next agent MUST read
# Files probably not necessary
# Open questions
```

A completely fresh Orchestrator should be able to resume from this plus referenced files.

---

# 25. Worker ASSIGNMENT.md

Suggested metadata:

```yaml
---
id: W-021-02
parent: O-021
type: implementation
status: assigned
base_commit: <commit>
---
```

Suggested content:

```markdown
# Objective
# Why this task exists
# Read first
# You own
# Read-only dependencies
# Do not modify
# Input contract
# Output contract
# Acceptance criteria
# Required tests
# Required runtime evidence
# Deliverables
# Escalation conditions
```

---

# 26. Worker RESULT.md

Suggested:

```markdown
# Result

Status: COMPLETE | PARTIAL | BLOCKED

## Summary
## Changes made
## Files modified
## Tests
## Typecheck / lint / build
## Runtime verification
## Screenshots / logs / evidence
## Acceptance criteria mapping
## Deviations from assignment
## Risks
## Unresolved questions
## Recommended follow-up
## Commit / branch / PR
```

Claims should be backed by evidence whenever feasible.

---

# 27. Single-writer rule

| Document | Writer |
|---|---|
| PROJECT.md | Project Lead |
| ROADMAP.md | Project Lead |
| STATE.md | Project Lead |
| Project DECISIONS.md | Project Lead |
| BRIEF.md | Project Lead |
| PLAN.md | Orchestrator |
| STATUS.md | Orchestrator |
| Task DECISIONS.md | Orchestrator |
| HANDOFF.md | Orchestrator |
| Worker ASSIGNMENT.md | Parent |
| Worker RESULT.md | Worker |
| Application code | Worker |
| Verification report | Verifier |

The coding agent should identify how much of this can be practically enforced.

---

# 28. Context exhaustion and replacement

Context management must be first-class.

Recommended checkpoint triggers:

- Worker completion;
- phase transition;
- important decision;
- context compaction/high usage;
- before task closure.

Suggested policy:

```text
No compaction
→ normal

First compaction
→ mandatory handoff checkpoint
→ agent may continue

Second compaction
→ mandatory checkpoint
→ prefer replacement

Repeated compaction
→ retire session
→ fresh agent resumes
```

The exact mechanism must be verified against current Cursor capabilities.

---

# 29. Fresh-agent resume flow

```text
O-021 session S1
   ↓
HANDOFF updated
   ↓
S1 retired
   ↓
O-021 session S2 starts
   ↓
reads:
BRIEF
PLAN
STATUS
HANDOFF
DECISIONS
relevant Worker results
   ↓
continues
```

The replacement should not require the old conversation transcript.

---

# 30. Runtime registry

Potential future machine-readable file:

```text
.agents/runtime/agents.json
```

Possible uses:

- current owner;
- duplicate-agent detection;
- stale-session detection;
- handoff support;
- hook enforcement.

Do not build unless benefits justify complexity.

---

# 31. Locks and ownership

Possible future locks:

```text
.agents/runtime/locks/
  creator-schema.lock
  billing-schema.lock
  project-state.lock
```

Potential behavior:

```text
O-021 wants creator schema change
   ↓
schema owned by O-023
   ↓
do not modify silently
   ↓
raise dependency to Project Lead
```

Evaluate whether documentation-only ownership is enough for v1.

---

# 32. Git isolation

Parallel implementation must not share one mutable working tree.

Investigate:

- Cursor Cloud Agent branches;
- Git worktrees;
- task branches;
- worker branches;
- PRs.

Possible conceptual structure:

```text
main
│
├── task/O-021-creator-search
│   ├── worker/W-021-01
│   ├── worker/W-021-02
│   └── worker/W-021-03
│
├── task/O-022-billing
│   ├── worker/W-022-01
│   └── worker/W-022-02
│
└── task/O-023-ingestion
    ├── worker/W-023-01
    └── worker/W-023-02
```

Determine whether this is actually necessary or whether a simpler branch model fits Cursor Cloud Agents better.

---

# 33. Integration ownership

Evaluate:

### Model A — Orchestrator-managed integration
### Model B — dedicated Integration Worker
### Model C — Worker PRs into task branch

Recommend one for Affitio.

---

# 34. Cursor Rules

Persistent rules should contain only durable invariants.

Possible examples:

```text
- Project Lead and Orchestrator normally do not edit application code.
- Workers do not edit global project state.
- Never silently expand assigned scope.
- Parallel Workers require explicit ownership boundaries.
- Shared contracts should be stabilized before parallel implementation.
- Implementation claims require evidence.
- Major architecture changes require escalation.
- Project-wide decisions must be promoted to canonical docs.
- Do not treat RESULT.md as proof without verification.
```

Avoid a giant rules file.

---

# 35. Cursor Skills

Potential reusable workflows:

```text
.cursor/skills/
├── project-lead/
├── create-task-brief/
├── orchestrate/
├── delegate-worker/
├── checkpoint/
├── handoff/
├── resume-task/
├── verify-work/
├── close-task/
└── archive-task/
```

Determine which should actually be Skills versus Rules, agents, scripts, or templates.

---

# 36. Cursor custom agents/subagents

Potential:

```text
.cursor/agents/
├── affitio-orchestrator.md
├── implementation-worker.md
├── investigation-worker.md
├── verifier.md
├── architecture-reviewer.md
└── security-reviewer.md
```

Determine:

- which roles should be custom agents;
- which should be Skills;
- whether Project Lead should be a Skill/custom mode rather than a child agent;
- how Cloud Agent execution changes the design.

---

# 37. Hooks and enforcement

Potential enforcement areas:

- Worker cannot edit global project state;
- Verifier cannot edit application code;
- Project Lead cannot edit product code;
- Worker cannot modify paths outside ownership;
- Worker completion triggers checkpoint;
- context compaction triggers HANDOFF requirement;
- task cannot close without verifier result;
- only Project Lead updates global state;
- only one global Project Lead has global write authority.

For every proposed control classify:

```text
SUPPORTED NOW
SUPPORTED WITH WORKAROUND
NOT SUPPORTED
NOT WORTH IMPLEMENTING
```

Verify current Cursor behavior. Do not assume.

---

# 38. Definition of Ready — Orchestrator task

```text
□ Goal defined
□ Business value understood
□ Scope defined
□ Non-goals defined
□ Acceptance criteria defined
□ Relevant architecture identified
□ Major dependencies identified
□ Major decisions resolved or explicitly open
□ Verification strategy understood
□ Ownership reasonably clear
```

---

# 39. Definition of Done — Worker

```text
□ Acceptance criteria addressed
□ Implementation complete
□ Tests updated where appropriate
□ Relevant tests pass
□ Typecheck passes
□ Lint passes
□ Relevant build passes
□ Runtime behavior verified where appropriate
□ No unrelated files changed
□ RESULT completed
□ Risks documented
□ Commit/branch/PR available
```

Adapt commands to actual Affitio tooling.

---

# 40. Definition of Done — Orchestrator

```text
□ Required Worker tasks complete
□ Worker output reviewed
□ Cross-Worker integration validated
□ Acceptance criteria verified
□ Independent verification performed
□ Relevant broader tests pass
□ Known risks documented
□ No unresolved blockers
□ STATUS updated
□ final HANDOFF/summary updated
□ durable decisions flagged to Project Lead
```

---

# 41. Definition of Done — Project Lead

```text
□ Business/product objective satisfied
□ Architecture remains coherent
□ Scope did not expand unintentionally
□ Important evidence reviewed
□ Durable documentation updated
□ Project STATE updated
□ ROADMAP updated if necessary
□ Important decisions promoted
□ Remaining risks communicated to user
```

---

# 42. Evidence-oriented delivery

Prefer evidence over self-report.

Examples:

- tests;
- typecheck;
- lint;
- build;
- API request/response;
- benchmark;
- screenshots;
- video;
- migration output;
- logs;
- database query;
- diff summary.

---

# 43. Runtime state vs committed state

Assess what should be committed.

Likely durable:

- PROJECT
- ROADMAP
- STATE
- DECISIONS
- BRIEF
- PLAN
- STATUS
- HANDOFF
- ASSIGNMENT
- RESULT

Likely ephemeral/gitignored:

- session IDs;
- transient locks;
- context percentage;
- hook flags;
- heartbeat files;
- temporary transcripts.

---

# 44. Human readability

Durable documents must remain understandable to the user.

Machine-readable metadata may supplement Markdown but should not replace clear summaries.

---

# 45. Naming and IDs

Possible:

```text
M-003    milestone
O-021    orchestrator task
W-021-01 worker
V-021-01 verifier
D-041    decision
R-012    risk
DEP-009  dependency
```

Use only as much bureaucracy as is helpful.

---

# 46. Status vocabulary

Keep small and consistent:

```text
PLANNED
READY
ACTIVE
BLOCKED
REVIEW
VERIFYING
COMPLETE
CANCELLED
SUPERSEDED
```

---

# 47. Monorepo awareness

Inspect:

- apps;
- packages;
- Cloudflare Workers;
- database packages;
- shared types;
- scripts;
- build graph;
- Turborepo configuration if applicable;
- deployment boundaries.

Do not assume folder boundaries equal safe ownership boundaries.

---

# 48. CI/CD integration

Map actual checks to roles.

Potential model:

```text
Worker:
targeted tests
affected package typecheck

Orchestrator:
integration/broader tests

Final:
normal repository CI
```

Avoid wasteful full-repo checks at every leaf if the build system supports affected scopes.

---

# 49. Security and destructive operations

Assess least-privilege access for agents.

High-risk operations should receive stronger escalation:

- production migrations;
- deleting infrastructure;
- payment changes;
- auth changes;
- breaking APIs;
- deleting persistent data;
- secret rotation.

Project-management and verifier agents generally should not need production secrets.

---

# 50. Architecture and business changes

Workers should not silently:

- replace database technology;
- introduce infrastructure;
- replace queue architecture;
- change pricing;
- change credits;
- alter subscription entitlement rules;
- modify retention/freshness policies;
- materially change customer-facing behavior.

Escalate instead.

---

# 51. Maturity levels

## Level 0 — documentation only
Roles, templates, manual delegation.

## Level 1 — Cursor configuration
Rules, Skills, custom agents.

## Level 2 — lightweight enforcement
Hooks, validation, closure checks.

## Level 3 — runtime coordination
Session registry, ownership enforcement, locks.

## Level 4 — advanced orchestration
Automated dispatch, agent replacement, dashboards.

Affitio should likely start between Level 0 and Level 1 and selectively adopt Level 2.

---

# 52. Anti-patterns

Avoid:

- one huge memory document;
- every agent editing global state;
- Workers receiving entire project history;
- agent self-certification;
- parallelism before contracts stabilize;
- two global Project Leads;
- preserving chat instead of decisions;
- excessive orchestration automation too early.

---

# 53. Example end-to-end workflow

```text
USER
  ↓
PROJECT LEAD
- discuss need
- challenge assumptions
- inspect state
- identify decisions
  ↓
create BRIEF
  ↓
ORCHESTRATOR
- inspect code
- validate assumptions
- create PLAN
- identify shared contracts
  ↓
Foundation Worker
  ↓
Parallel Workers
  ↓
Orchestrator review
  ↓
Verifier
  ↓
Fix Worker if needed
  ↓
Verifier PASS
  ↓
Orchestrator final STATUS/HANDOFF
  ↓
Project Lead final review
  ↓
promote durable decisions
update STATE/ROADMAP
  ↓
USER receives outcome, evidence, risks, next step
```

---

# 54. Coding-agent investigation tasks

Before proposing implementation, inspect the repository for:

## Existing agent configuration

Find:

- `AGENTS.md`;
- `.cursor/`;
- rules;
- Skills;
- agents;
- hooks;
- MCP/plugin configuration;
- existing agent instructions.

## Existing documentation

Find:

- architecture docs;
- product requirements;
- ADRs;
- implementation plans;
- operations docs;
- backlog/roadmap equivalents.

Identify overlap. Avoid duplicate sources of truth.

## Existing repository architecture

Map:

- apps;
- packages;
- Workers;
- databases;
- shared types;
- build system;
- testing;
- CI;
- deployment.

## Existing Git workflow

Determine:

- branching conventions;
- PR conventions;
- CI triggers;
- protected branches;
- merge strategy;
- Cloud Agent behavior.

## Existing quality gates

Identify actual commands for:

- lint;
- typecheck;
- unit tests;
- integration tests;
- E2E tests;
- build;
- deployment preview;
- security checks.

---

# 55. Cursor capability validation

Research and verify against the current environment/version:

- project Rules;
- `AGENTS.md`;
- Skills;
- custom Agents/Subagents;
- Cloud Agent behavior;
- worktrees;
- branches;
- hooks;
- lifecycle events;
- context compaction events if exposed;
- subagent completion events;
- stop/completion events;
- ability to inject follow-up instructions;
- ability to block tool/file actions;
- session/conversation IDs;
- transcript access;
- Cloud Agent hook support.

For each:

| Capability | Available? | Exact mechanism | Limitations | Recommendation |
|---|---|---|---|---|

---

# 56. Practical enforcement study

Determine whether these can be technically enforced:

1. Worker cannot modify `.agents/project/**`
2. Verifier cannot modify application code
3. Project Lead cannot modify application code
4. Worker cannot modify paths outside assignment
5. Orchestrator must update HANDOFF after compaction
6. Orchestrator cannot close without verifier result
7. Only Project Lead can update global project state
8. Only one global Project Lead can be active

For each:

```text
Possible?
Mechanism?
Reliability?
Complexity?
Worth implementing now?
```

---

# 57. Context-management study

Explicitly answer:

1. How does Cursor expose context usage?
2. Does it expose pre-compaction events?
3. Can hooks act before/after compaction?
4. Can hooks block later actions?
5. Can hooks inject a mandatory follow-up?
6. Can replacement Cloud Agents be launched programmatically?
7. Can session IDs be persisted?
8. Can a new agent reliably resume from files?

Recommend the simplest reliable design.

---

# 58. Concurrency study

Determine how many agents can safely operate given:

- repository hotspots;
- Cloud Agent environments;
- Git strategy;
- worktrees;
- CI capacity;
- shared schemas;
- migrations;
- root workspace files.

Identify:

- safe parallel examples;
- unsafe parallel examples;
- conflict-heavy modules;
- foundation work that should remain sequential.

---

# 59. Recommended initial implementation

Propose a minimal v1.

Likely include:

```text
AGENTS.md / core rules
.agents/project/
.agents/runs/
task templates
Project Lead workflow
Orchestrator workflow
Worker workflow
Verifier workflow
basic Cursor Skills/Agents
manual checkpoint/handoff
Git isolation guidance
```

Likely defer unless clearly justified:

```text
complex locks
agent registry automation
automatic agent replacement
custom dashboard
large hook framework
distributed task engine
```

---

# 60. Suggested templates

If appropriate, propose:

```text
.agents/templates/
├── PROJECT.template.md
├── ROADMAP.template.md
├── STATE.template.md
├── DECISION.template.md
├── BRIEF.template.md
├── PLAN.template.md
├── STATUS.template.md
├── HANDOFF.template.md
├── ASSIGNMENT.template.md
├── RESULT.template.md
└── VERIFICATION.template.md
```

Avoid unnecessary proliferation.

---

# 61. Potential scripts

Potential later utilities:

```text
scripts/agents/create-task
scripts/agents/create-worker
scripts/agents/check-handoff
scripts/agents/check-task-done
scripts/agents/list-active
scripts/agents/archive-task
```

Do not implement unless they clearly improve the workflow.

---

# 62. Machine-readable metadata

Consider YAML front matter for task files.

Example:

```yaml
---
id: O-024
type: orchestrator
title: Contact Discovery MVP
status: active
owner: orchestrator
depends_on:
  - O-021
---
```

Assess benefits versus synchronization/bureaucracy costs.

---

# 63. State consistency

Avoid duplicating mutable facts across many files.

Bad:

```text
STATE: ACTIVE
PLAN: COMPLETE
HANDOFF: BLOCKED
agents.json: REVIEW
```

Define canonical ownership.

Example:

```text
Task status canonical → STATUS metadata
Project STATE → Project Lead summary
```

---

# 64. User interaction model

The user should normally be able to say:

```text
"I want to build X."
```

Project Lead should:

```text
understand
challenge
clarify
plan
delegate
review
report
```

The user should not need to manually coordinate every Worker.

Support both:

- automatic delegation by parent agent;
- manual Cloud Agent launch by user.

Both should use the same repository task contract.

---

# 65. Launch-prompt portability

A task should ideally launch with a tiny prompt:

```text
You are the Orchestrator for O-024.

Read:
.agents/runs/O-024-contact-discovery/BRIEF.md

If resuming, read HANDOFF.md first.

Follow the Affitio Orchestrator operating rules.
```

The repository should contain the context, not a giant pasted prompt.

---

# 66. Replacement portability

Likewise:

```text
Resume O-024.

Read HANDOFF first, then STATUS, PLAN, BRIEF, DECISIONS,
and only Worker results referenced by HANDOFF.

Do not assume previous chat context.
```

---

# 67. Review distinction

### Verifier asks:

> Does this implementation actually satisfy the technical acceptance criteria?

### Project Lead asks:

> Does this completed work still satisfy the intended product goal, architecture, dependencies, and project direction?

Do not collapse these into one review.

---

# 68. Acceptance-criteria traceability

Worker RESULT can map evidence:

| Acceptance criterion | Evidence | Result |
|---|---|---|
| Country filter works | integration test X | PASS |
| Subscriber range works | API test Y | PASS |
| No cross-DB join | code inspection | PASS |
| Performance target | benchmark Z | PASS/UNKNOWN |

---

# 69. Escalation conditions

Worker should escalate when:

- assignment conflicts with architecture;
- scope must expand;
- shared contract is wrong;
- dependency is missing;
- destructive migration is required;
- product behavior is ambiguous;
- security concern appears;
- billing behavior changes;
- forbidden ownership must be modified.

Orchestrator escalates to Project Lead.

Project Lead determines whether user input is required.

---

# 70. Cost and efficiency

The process must remain proportionate.

Do not use full:

```text
Project Lead → Orchestrator → Worker → Verifier
```

for a typo.

Routing should consider:

- complexity;
- risk;
- ambiguity;
- parallelism opportunity;
- cross-module impact;
- reversibility.

---

# 71. Suggested routing heuristic

### Direct / one Worker

- one module;
- low risk;
- clear acceptance criteria;
- little architecture impact.

### Orchestrator

- multiple modules;
- multiple Workers;
- dependencies;
- multi-stage implementation;
- integration risk.

### Project Lead + additional reviewers

- major architecture;
- auth/security;
- billing/payments;
- migrations;
- large product behavior changes.

---

# 72. Application to Affitio domains

The study should consider current Affitio areas.

## Creator search

Possible slices:

- query service;
- UI;
- filters;
- tests.

Shared search/API contracts likely need stabilization first.

## YouTube ingestion

Possible slices:

- browser scraping;
- retry handling;
- queues;
- logging;
- persistence.

## Billing and usage

Higher-risk:

- Stripe;
- entitlements;
- Search Credits;
- Contact Credits;
- Agent Tokens;
- reconciliation.

## Creator database

Cross-cutting:

- ingestion;
- search;
- contact discovery;
- analytics.

Schema ownership may require explicit coordination.

## Agentic affiliate management

Likely large and multi-phase, well suited to dedicated Orchestrator(s), Workers, verification, and durable decisions.

---

# 73. Required study deliverables

## Deliverable 1 — Current-state assessment

Explain:

- repo structure;
- agent configuration;
- documentation system;
- Git/CI workflow;
- constraints.

## Deliverable 2 — Fit-gap analysis

| Component | Existing equivalent | Gap | Recommendation |
|---|---|---|---|
| Project memory | ... | ... | ... |
| Task workspace | ... | ... | ... |
| Rules | ... | ... | ... |
| Skills | ... | ... | ... |
| Workers | ... | ... | ... |
| Verifier | ... | ... | ... |
| Handoff | ... | ... | ... |
| Hooks | ... | ... | ... |

## Deliverable 3 — Affitio-specific target architecture

Show:

- final folder structure;
- authority model;
- file ownership;
- agent definitions;
- Skill definitions;
- hook architecture;
- branch/worktree strategy;
- context-management strategy.

## Deliverable 4 — Minimum viable implementation plan

Split into phases.

Example:

```text
Phase A — documents/templates
Phase B — Rules and agent definitions
Phase C — handoff/checkpoint workflow
Phase D — lightweight hooks
Phase E — evaluate automation
```

For each include:

- files changed;
- behavior;
- dependencies;
- risks;
- verification.

## Deliverable 5 — Concrete proposed files

Provide draft contents or diffs for:

- AGENTS.md additions;
- Project Lead Skill/Agent;
- Orchestrator agent;
- Worker agent;
- Verifier agent;
- templates;
- rules;
- hooks if recommended.

Do **not** apply major changes unless explicitly authorized.

## Deliverable 6 — Capability verification

Document which Cursor capabilities were verified and their limitations.

## Deliverable 7 — Example simulation

Use one realistic, bounded Affitio task to demonstrate:

```text
user request
→ Project Lead
→ BRIEF
→ Orchestrator PLAN
→ Worker assignments
→ Worker results
→ verification
→ Project Lead closure
```

This is a workflow simulation; it does not need to implement the feature.

---

# 74. Questions the coding agent must explicitly answer

1. Does Affitio already have an equivalent of `AGENTS.md`?
2. What existing documentation overlaps with `PROJECT.md`?
3. Should `ROADMAP.md` exist separately from current plans?
4. Should task runs be committed to Git?
5. Where should completed task runs be archived?
6. Should STATUS be manually written or generated?
7. Should YAML front matter be used?
8. What is the simplest reliable Worker ownership mechanism?
9. Can current Cursor hooks enforce path restrictions?
10. Can current Cursor hooks detect context compaction?
11. Can a hook inject a required follow-up?
12. Can a hook prevent further tool actions until HANDOFF is updated?
13. Can Cloud Agents use project hooks?
14. Can a replacement agent be automatically launched?
15. Is automatic replacement desirable?
16. How should branches/worktrees be organized?
17. Should Workers open PRs?
18. Who performs integration?
19. How should CI checks be divided?
20. How should Project Lead review be represented?
21. How do we prevent two global Project Leads from writing state?
22. Is a runtime registry worth implementing now?
23. Are lock files worth implementing now?
24. How do we prevent stale control docs?
25. How much should remain process rather than automation?
26. What is the smallest useful v1?
27. What complexity should be explicitly postponed?

---

# 75. Design-priority hierarchy

When tradeoffs arise, optimize in this order:

1. **Correctness**
2. **Human understandability**
3. **Recoverability**
4. **Clear ownership**
5. **Safety**
6. **Parallel speed**
7. **Automation sophistication**

Do not sacrifice clarity or correctness just to maximize concurrency.

---

# 76. Simplicity principle

This architecture can easily become too complicated.

Therefore:

> Every mechanism must justify its existence through a real failure mode it prevents or meaningful efficiency it creates.

Examples:

Good reason for HANDOFF:
- context really is lost.

Good reason for branch isolation:
- parallel agents really do edit code.

Weak early reason for a lock service:
- "maybe someday agents could conflict."

Prefer the simplest mechanism that solves a demonstrated problem.

---

# 77. Recommended initial philosophy

Start with:

```text
Markdown
Git
Cursor Rules
Cursor Skills
Cursor agents/subagents
Cloud Agent isolation
CI
```

Then observe actual failure modes.

Only automate recurring problems.

---

# 78. Final target experience

The user talks primarily to the Project Lead.

The Project Lead understands the project, challenges missing assumptions, prepares tasks, and delegates.

Multiple Orchestrators can run at the same time.

Each Orchestrator manages its own bounded task and Workers.

Workers perform implementation.

Verifiers independently check important work.

Repository documents preserve enough context that any agent can be replaced.

Cross-task dependencies are controlled by the Project Lead.

Global project state has one writer.

The system allows high parallelism without losing architectural coherence.

---

# 79. Final instruction to the coding agent

Study the actual Affitio repository before making recommendations.

Do not mechanically reproduce this proposed structure.

Your goal is to determine:

> **What is the simplest, safest, and most maintainable implementation of this Project Lead → Orchestrator → Worker operating model for the real Affitio repository and Cursor Cloud Agent workflow?**

Preserve existing good architecture and conventions.

Explicitly identify assumptions.

Verify Cursor capabilities.

Challenge this design where it is unnecessarily complex.

Recommend what to implement now, what to defer, and why.

Produce a detailed Markdown report that a human can review before implementation.

Where useful, include:

- diagrams;
- example file trees;
- example configuration;
- proposed templates;
- pseudocode;
- hook designs;
- branch workflows;
- concrete references to the Affitio codebase.

The purpose is not to build the most sophisticated orchestration platform.

The purpose is to create a development system that lets the user confidently delegate meaningful Affitio work to multiple AI agents while retaining control, continuity, safety, and understandable project state.
