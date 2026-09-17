Worked for 37s

Yes — in that environment I would not make “one clone per agent” the primary design. If the devbox assumes one fixed repo path for the dev server, browser preview, ports, auth callbacks, etc., multiple clones can fight the platform instead of helping.

The pattern I would seriously consider is:

> One canonical/routed working tree + parallel scoped editors + exactly one Git controller.



1. Let agents edit the same working tree, but never let them control Git

For example:

/devbox/project   ← fixed path expected by devbox
      │
      ├── Agent A → apps/web/search/**
      ├── Agent B → packages/creator/**
      ├── Agent C → workers/ingest/**
      │
      └── Integration Agent
             │
             └── ONLY agent allowed to:
                  git add
                  git commit
                  git switch
                  git reset
                  git clean
                  git rebase
                  package install
                  migrations

The coding agents become essentially filesystem editors.

They can use:

git diff
git status
git show
git log

but should not use Git commands that mutate repository state.

That distinction is extremely important.


---

2. Give every task a write lease

Your orchestrator could maintain:

AFF-101:
  agent: agent-a
  base_sha: abc123
  write:
    - apps/web/src/features/search/**
    - packages/search/**

AFF-102:
  agent: agent-b
  base_sha: abc123
  write:
    - workers/creator-ingest/**

AFF-103:
  agent: agent-c
  base_sha: abc123
  write:
    - apps/web/src/features/campaign/**

Before Agent A edits a file:

apps/web/src/features/search/SearchPage.tsx

the orchestrator checks:

Does AFF-101 own this path?

YES → edit allowed
NO  → block / request scope expansion

You can therefore have:

Agent A ──────┐
Agent B ──────┼──→ Same filesystem
Agent C ──────┘
                    │
                    ▼
               Dev server
                    │
                    ▼
               Browser/UI

This actually has one nice property for your situation:

the existing UI continues to work normally.

Hot reload also sees all changes.


---

3. But don't have each agent run git commit

This is where a shared checkout usually breaks down.

Imagine:

Agent A:
git add apps/web/...
git commit

Agent B simultaneously:
git add workers/...
git commit

Both manipulate:

.git/index
HEAD
refs

You can get races or one agent accidentally committing another agent's files.

Instead:

Workers
   ↓
edit files only
   ↓
Integration Orchestrator
   ↓
captures each task independently
   ↓
creates PR branches

So you have one Git writer.


---

There's a useful Git trick here

You don't necessarily have to switch branches to create the separate PR commits.

Git supports using a different index file.

Conceptually:

Working tree
 ├── changes from A
 ├── changes from B
 └── changes from C

       ↓

temporary index A
       ↓
PR A commit

temporary index B
       ↓
PR B commit

temporary index C
       ↓
PR C commit

For example, say:

BASE=abc123
TASK=AFF-101
INDEX=/tmp/index-$TASK

Start an index representing exactly the base commit:

rm -f "$INDEX"

GIT_INDEX_FILE="$INDEX" \
git read-tree "$BASE"

Then add only Agent A's owned paths:

GIT_INDEX_FILE="$INDEX" \
git add -A -- \
  apps/web/src/features/search \
  packages/search

Write a Git tree:

TREE=$(
  GIT_INDEX_FILE="$INDEX" \
  git write-tree
)

And construct a commit:

COMMIT=$(
  echo "AFF-101: implement creator search" |
  git commit-tree "$TREE" -p "$BASE"
)

Then create/update its branch ref:

git update-ref \
  refs/heads/agent/AFF-101 \
  "$COMMIT"

Notice what we didn't do:

git switch
git checkout
git reset

Your current working tree stays exactly where it is.

Your dev server keeps running.

And you now have:

agent/AFF-101

containing only AFF-101's files.

That's potentially very useful in your devbox constraint.


---

So the architecture becomes

Project Lead
                        │
                        ▼
                Implementation
                 Orchestrator
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
          Agent A    Agent B    Agent C
          AFF-101    AFF-102    AFF-103
             │          │          │
             │    filesystem only  │
             └──────────┼──────────┘
                        ▼
              CANONICAL WORKSPACE
              /devbox/project
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        Dev server             Git Controller
             │                     │
             ▼              temporary indexes
        UI validation                │
                               ┌─────┼─────┐
                               ▼     ▼     ▼
                              PR A  PR B  PR C

I think this is much closer to what your environment needs.


---

There is one important downside

Your live devbox will contain:

A + B + C

simultaneously.

Therefore when you look at the UI, you're testing:

PR A + PR B + PR C

rather than:

PR A only

This means you need to distinguish two types of validation.

Development validation

During parallel coding:

combined workspace
A + B + C

Useful for:

TypeScript errors

hot reload

UI appearance

smoke tests

obvious integration problems


PR validation

Before PR A is considered ready:

base + A only

This needs isolated validation.

And because you only have one routed UI environment, that part probably has to be serialized.

That's okay.

You don't necessarily need:

> parallel coding + parallel UI validation.



You can have:

> parallel coding + serialized final UI validation.



That's still a huge throughput improvement.


---

I would create an Integration Queue

Suppose three agents finish around the same time.

Coding

A ████████████ done
B ███████████████ done
C ██████████ done

Instead of fighting over the UI environment:

Integration Queue

AFF-103 ──┐
AFF-101 ──┼──→ PR-isolated validation
AFF-102 ──┘

The orchestrator validates:

Base + AFF-103
      ↓
lint
typecheck
tests
UI/browser validation
      ↓
PR ready

Base + AFF-101
      ↓
...

Base + AFF-102
      ↓
...

Parallelize the expensive human/agent implementation time.

Serialize the comparatively short integration step.


---

Another option: shadow directories, not clones

If having several agents modifying /devbox/project directly worries you, there's an intermediate model.

/devbox/project             ← canonical routed workspace

/devbox/.agents/AFF-101/
                    search/
                    ...

/devbox/.agents/AFF-102/
                    ingest/
                    ...

Agent A works in:

/devbox/.agents/AFF-101

Agent B works in:

/devbox/.agents/AFF-102

They don't need complete Git repos.

They could just have copies of the relevant portions of the source tree.

When the agent wants UI validation:

Agent sandbox
     ↓
generate patch
     ↓
Orchestrator
     ↓
apply into /devbox/project
     ↓
UI validation

For example:

AFF-101.patch
AFF-102.patch
AFF-103.patch

The fixed routed workspace remains your execution environment.

This is safer than multiple agents modifying the same filesystem, but UI feedback becomes more serialized.


---

There's also a hybrid I think would work particularly well

I would probably use this for your agent architecture.

Normal development

Agents directly edit non-overlapping files in the canonical workspace:

Agent A
  └ search/**

Agent B
  └ campaign/**

Agent C
  └ ingest/**

You get immediate:

hot reload
TypeScript diagnostics
browser UI
runtime feedback

When a task reaches a checkpoint

The orchestrator snapshots its owned files:

AFF-101 snapshot
base: 82ab991
paths:
  apps/web/features/search/**
  packages/search/**

using either:

git diff / patch

or the alternate-index approach above.

Agent A can then continue working or stop.

Finalization

The Git controller creates:

agent/AFF-101
agent/AFF-102
agent/AFF-103

without letting the workers touch Git state.

Integration

The orchestrator figures out dependencies:

AFF-101 ─────┐
             ├── independent
AFF-102 ─────┘

AFF-103
   │
   └── depends on AFF-101

Then CI + UI integration happens accordingly.


---

I'd also classify certain files as exclusive

Some files should never participate in parallel editing.

For example:

exclusive:
  - package.json
  - pnpm-lock.yaml
  - turbo.json
  - tsconfig.json
  - migrations/**
  - schema/**
  - wrangler.jsonc
  - .github/**
  - shared API contracts
  - generated files

If Agent A needs:

pnpm-lock.yaml

the orchestrator assigns:

GLOBAL LOCK: dependency-change
owner: AFF-101

Agent B must wait or split its dependency change into another task.

You can therefore have two locking levels:

PATH LEASE
apps/web/search/**

RESOURCE LEASE
dependency-install
database-schema
migration
API-contract
CI-config

The latter is important because two tasks can touch different files but still manipulate the same conceptual resource.


---

One more thing I would prohibit

Agents should not independently run commands such as:

pnpm install
npm install
git clean
git reset
git checkout
git switch
git stash
git commit
git rebase
git merge

in this architecture.

Instead:

Worker Agent
    │
    ├── edit
    ├── lint
    ├── test scoped package
    └── inspect

Git/Environment Controller
    │
    ├── dependencies
    ├── migrations
    ├── branch creation
    ├── commits
    ├── reset
    └── PR creation

This gives you a much stronger safety boundary.


---

What I think fits your devbox best

I would change my previous recommendation to this:

ONE DEVBOX
                     │
              ONE ROUTED REPO
                     │
              /workspace/affitio
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
   Worker A       Worker B      Worker C
   scope A         scope B       scope C
       │             │             │
       └─────────────┼─────────────┘
                     │
                     ▼
               shared dev UI
                     │
                     ▼
               Git Controller
                     │
           alternate indexes /
              task snapshots
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      PR 101       PR 102       PR 103
        │            │            │
        └────────────┼────────────┘
                     ▼
             serial UI gate
                     │
                     ▼
                 merge queue

So the guiding principle becomes:

> Don't isolate the repository. Isolate authority.



Workers can share the routed filesystem provided their write scopes do not overlap. Only one orchestrator controls Git state and global resources. Then use temporary Git indexes/task snapshots to materialize separate PRs without constantly switching the devbox's active branch.

That is probably the avenue I'd explore first for the Project Lead → Orchestrator → Worker architecture you're building.