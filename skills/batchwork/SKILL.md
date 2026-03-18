---
name: batchwork
description: Multi-project batch work router. maintenance (audit/fix), push-work (advance TODOs), push-beyond (discover features → ROI → prototypes).
argument-hint: "<mode> [projects...] [--all] [--scope P0|P0-P1|all] [--dry-run] [--resume] [--top N]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Skill, Agent, TaskCreate, TaskUpdate, TaskList, TaskGet, AskUserQuestion, WebSearch, WebFetch
---

# Batchwork v5.3

## Flexibility Principles

The phases and pipelines below are **defaults, not mandates**. Use judgment and adapt to the situation:

- **Narrow scope to match intent.** If the user says "check security in pact's contracts," only scan security categories in that directory — don't run all 14 categories across every file.
- **Skip phases that add no value.** Single project? Skip Phase 3 (Cross-Project). User already has context? Shorten or skip Phase 1. User wrote their own PLAN.md? Go straight to execution via `--resume`.
- **Scale agents to scope.** A focused scan of one directory doesn't need 3 agents. A single obvious fix doesn't need a full research phase. Match the machinery to the job.
- **Accept any reasonable PLAN.md.** The numbered/categorized format is ideal, but any checklist with `- [ ]` items and enough description to act on works — especially user-authored plans.
- **Skip the user gate when intent is clear.** If the invocation already specifies exactly what to do (e.g., "maintenance pact --scope P0"), don't ask again — proceed.
- **Push-beyond can start at any stage.** User already knows what to build? Skip Discover and Evaluate — use their idea as the spec and go straight to prototyping.
- **Adapt the pipeline, don't fight the user.** If the request doesn't fit neatly into a mode's pipeline, reshape the pipeline to serve the request — not the other way around.

## Modes

| Mode | Pipeline | Key Question |
|------|----------|-------------|
| **maintenance** | Scan → Fix (Phases 1-5) | "What's broken?" |
| **push-work** | Scan → Fix (Phases 1-5) | "What's unfinished?" |
| **push-beyond** | Discover → Evaluate → Prototype (Stages 1-4) | "What should we build next?" |

## Flags

| Flag | Default | Applies To | Description |
|------|---------|------------|-------------|
| `--scope` P0/P0-P1/all | `all` | maintenance, push-work | Filter by priority |
| `--dry-run` | `false` | all | Stop before execution |
| `--resume` | `false` | all | Resume from existing output |
| `--top N` | `3` | push-beyond | Max prototypes per project |

## Project Resolution

1. Resolve names to `~/projects/{name}/`
2. Validate each is a git repo
3. `--all` → scan `~/projects/` for all `.git/` directories
4. Abort with error if any path is invalid

## Output Directory

All output goes to `{project}/.batchwork/{mode}/` (namespaced by mode to prevent conflicts):
- maintenance: `RESEARCH.md`, `PLAN.md`
- push-work: `RESEARCH.md`, `PLAN.md`
- push-beyond: `IDEAS.md`, `PROPOSALS.md`
- Cross-project: `~/projects/.batchwork/{mode}/`

## Supporting Files

Prompt templates live in [prompts/](prompts/) — loaded on demand, not on every invocation:
- [maintenance-agent.md](prompts/maintenance-agent.md) — Phase 1 research agents
- [discover-agent.md](prompts/discover-agent.md) — Stage 1 discovery agents
- [prototype-agent.md](prompts/prototype-agent.md) — Stage 3 builder agents

Substitute `{project}`, `{mode}`, `{file_list}` etc. before passing to agents.

## Orchestration: Task Tracking

Use tasks for visibility across long-running operations:

```
On start:  TaskCreate per project — "{mode}: {project}" status pending
Phase/Stage start: TaskUpdate → in_progress with activeForm
Phase/Stage end:   TaskUpdate → completed with metadata (findings count, PR URLs, etc.)
```

Users can run TaskList anytime to check progress.

## Parallelism Rules

**To run agents in parallel, make multiple Agent tool calls in a SINGLE message.** Sequential messages = sequential agents.

```
WRONG:                              RIGHT:
Message 1: Agent(alpha)             Message 1: Agent(alpha)
  ← wait                                       Agent(pact)
Message 2: Agent(pact)                          Agent(smalltalk)
  ← wait                              ← all return together
```

**Concurrency budget**: Max ~10 agents per batch. If more needed, batch in waves:
- 6 projects × 2 agents = 12 → split into 2 waves of 6
- 6 projects × 5 agents = 30 → split into 3 waves of 10

**Context exhaustion**: If an agent fails due to context limits, retry once with half the file list.

---

# MAINTENANCE & PUSH-WORK (Phases 1-5)

```
Phase 1: Research + Plan       → Parallel Explore agents → RESEARCH.md + PLAN.md
            ── USER GATE ──
Phase 2: Execute via /batch    → Skill("batch") per project → worktrees → PRs
Phase 3: Cross-Project         → Shared patterns from cross-project PLAN.md
Phase 4: Config Refresh        → CLAUDE.md, .claude/, memory (propose changes only)
Phase 5: Verification          → Tests, builds, final report
```

## Phase 1 — Research + Plan

### Categories

These are the full category lists. **Scan only the categories relevant to the user's intent** — if they ask about security, don't also scan Code Quality. When no focus is specified, use all.

**maintenance (14)**: Security, Reliability, Financial Calculations, Race Conditions, Data Integrity, API Contracts, Environment & Config, Performance, Error Handling, Type Safety, Database, Auth & Authorization, Dependencies, Code Quality.

**push-work (5)**:
1. **In-Code TODOs** — grep for `TODO|FIXME|HACK|XXX`, group by intent
2. **GitHub Issues** — `gh issue list --state open` if gh CLI available
3. **Stubbed Functions** — "not implemented" throws, empty bodies, mock returns
4. **Test Gaps** — untested paths, missing test files, `skip`/`xit`/`xtest`
5. **Dead Ends** — functions >10 params with no docstring, unused exports

### Agent Execution

Read and substitute [maintenance-agent.md](prompts/maintenance-agent.md).

**Scope assignment** (prevents "Prompt is too long" errors). Scale agent count to the scope — a focused scan of one directory may only need 1 agent:
- **Small projects (<50 source files):** 1-2 agents, split by directory
- **Medium projects (50-200 files):** 2-3 agents by top-level directory groupings (e.g., Agent 1 = app/ + components/, Agent 2 = lib/ + utils/, Agent 3 = scripts/ + config)
- **Large projects (200+ files, monorepos):** Assign agents by package/app boundary (e.g., Agent 1 = apps/web/, Agent 2 = apps/api/, Agent 3 = packages/shared/)
- **Focused scope** (user specified directories/categories): May only need 1 agent for the targeted area

**Rules:**
- Avoid passing individual file paths for projects with >50 files — use directory scopes
- Use relative paths from project root (e.g., `app/`, not `/home/user/projects/foo/app/`)
- Keep the category list short — use names only, not full descriptions (see maintenance-agent.md)
- Total prompt should stay under ~2000 words

Spawn agents in a single message for parallelism. If total agents across all projects > 10, batch in waves.

```
# Example: 3 projects, 2 agents each = 6 agents → 1 wave
Single message with 6 Agent tool calls:
  Agent(subagent_type="Explore", prompt="...alpha group 1...")
  Agent(subagent_type="Explore", prompt="...alpha group 2...")
  Agent(subagent_type="Explore", prompt="...pact group 1...")
  Agent(subagent_type="Explore", prompt="...pact group 2...")
  Agent(subagent_type="Explore", prompt="...smalltalk group 1...")
  Agent(subagent_type="Explore", prompt="...smalltalk group 2...")
```

**Priority**: P0 (data loss/security/crash) → P1 (likely bug) → P2 (code smell) → P3 (nice-to-have).

### Output

**RESEARCH.md** — findings with file:line, evidence, impact.
**PLAN.md** — checklist for `/batch`: `- [ ] #001 [{Category}] {Title}` with steps and dependencies. This format is recommended but not required — any `- [ ]` checklist with actionable descriptions works.

Group items by independence for parallelism. Flag `Depends on: #001` where needed.

### User Gate

Skip the gate if the user's invocation already makes their intent clear (specific scope, specific project, clear direction). Otherwise:

1. Summary table: P0/P1/P2/P3/Total per project
2. Estimated agent load
3. Files created
4. `AskUserQuestion`: "Yes, all" / "P0 only" / "P0-P1 only" / "Stop here"
5. `--dry-run` stops here

## Phase 2 — Execute via /batch

Invoke per project using the Skill tool:

```
Skill("batch", args="Execute the plan in {project}/.batchwork/{mode}/PLAN.md.
PLAN.md + RESEARCH.md are the source of truth. Do NOT re-research.
Priority order (P0 first). Scope: {scope}.
Mark items done: change '- [ ]' to '- [x]' after each item.
Commit after EACH completed item — do not batch commits.
If tests fail: revert, mark (REVERTED).
Each work item or group → one PR.")
```

**Commit strategy** (prevents uncommitted changes on context exhaustion):
- Commit immediately after each item is implemented + tested
- Commit the PLAN.md checkbox update separately
- Never wait until end of batch to commit

Run `/batch` for all projects in parallel (one Skill call per project, all in single message).

## Phase 3 — Cross-Project

Skip this phase if only one project is involved or there are no cross-project dependencies.

Address items from `~/projects/.batchwork/{mode}/PLAN.md`.

- **Sequential** (order matters): API contracts → provider first then consumers. Shared types → source then dependents.
- **Parallel** (independent): Dep alignment, config standardization → one agent per project in single message.

## Phase 4 — Config Refresh

Skip if changes were minor or only touched one area. For each touched project:
1. **CLAUDE.md** — verify tech stack, structure, env vars. Keep under 200 lines.
2. **.claude/** — check stale settings, outdated skills/rules.
3. **Global memory** — update if project info changed.
4. **Skill improvements** — write to `~/.claude/skills/batchwork/SKILL_IMPROVEMENTS.md`. Do NOT edit SKILL.md directly.

## Phase 5 — Verification

1. Cross-project contract checks (API types, shared deps, env vars)
2. Run tests/builds on main if PRs merged
3. Final report: Tests/Build/Types/PR/Status per project
4. Offer to fix failures or flag for manual review

---

# PUSH-BEYOND (Stages 1-4)

Completely separate pipeline. Asks "what NEW thing should we build?" not "what's broken?"

**Start at whatever stage makes sense.** If the user already has ideas, skip Stage 1. If they already have a ranked list, skip to Stage 3. If they hand you a specific feature to prototype, go straight there.

```
Stage 1: Discover         → 1 general-purpose agent per project → IDEAS.md
Stage 2: Evaluate          → Main orchestrator scores ROI → PROPOSALS.md
            ── USER GATE ──
Stage 3: Prototype         → 1 builder agent per prototype (worktree) → branch + PR
Stage 4: Report            → Final table + recommendations
```

## Stage 1 — Discover

Read and substitute [discover-agent.md](prompts/discover-agent.md).

Spawn **one general-purpose agent per project** (not Explore — needs WebSearch + Bash for git). All agents in a single message for parallelism. Max 10 per wave.

### Discovery Dimensions

| Dimension | Sources |
|-----------|---------|
| **Project Identity** | README, CLAUDE.md, package.json |
| **Momentum** | `git log --oneline -30`, recent PRs |
| **Author Intent** | TODOs, roadmap files, GitHub issues |
| **Competitive Landscape** | Web search for 2-3 competitors |
| **User Surface** | UI pages, API endpoints, CLI commands |

**Cross-mode awareness**: If `.batchwork/maintenance/PLAN.md` exists, agents read it to avoid proposing features that conflict with unresolved P0 bugs.

### Output: IDEAS.md

```markdown
# {Project} — Push-Beyond Ideas
Generated: {date}

## Project Summary
## Recent Momentum
## Competitor Landscape

## Ideas
### #01 {Title}
- **Category**: {New Capability | Integration | Automation | Data/Analytics | UX/UI | Monetization}
- **One-liner**: {description}
- **User value**: {why users want this}
- **Existing leverage**: {reusable code/data/infra}
- **Effort hints**: {what's missing, new deps needed}
- **Scope**: {Small | Medium | Large}
- **Competitor ref**: {who does this}
```

## Stage 2 — Evaluate

Main orchestrator (no agents). Read each IDEAS.md + skim project codebase to verify effort hints.

### Scoring

**Impact (1-5)**: 5=core daily workflow/revenue, 4=significant weekly value, 3=nice-to-have, 2=niche, 1=internal-only.

**Effort (1-5)**: 1=<50 LOC reuse existing, 2=50-200 LOC one new dep, 3=200-500 LOC new patterns, 4=500-1000 LOC new infra, 5=1000+ LOC major arch change.

**ROI = Impact / Effort** (higher is better).

### Process

1. Read IDEAS.md per project
2. Quick codebase scan to verify effort hints (check if mentioned packages/APIs exist)
3. Score Impact and Effort per idea
4. Sort by ROI descending, cap at 10 per project
5. Write detailed prototype spec for top `--top N` (default 3)

### Output: PROPOSALS.md

```markdown
# {Project} — Feature Proposals
Generated: {date} | Ideas: {count} | Prototyping: top {N}

## Rankings
| # | Feature | Impact | Effort | ROI | Category | Scope |

## Prototype Specs
### #1 — {Title} (ROI: {score})
- **What**: {description}
- **Why now**: {why feasible with current codebase}
- **Prototype scope**: {MVP — max 200 LOC}
- **Entry point**: {where to wire in}
- **Dependencies**: {new packages/APIs, if any}
- **Smoke test**: {how to verify}
- **Not in prototype**: {what production version adds}
```

### User Gate

1. Proposals summary table: Project / Ideas / Top Pick / ROI / Scope
2. Estimated prototype load
3. `AskUserQuestion`: "Which to prototype?" → user picks, or "All top N" / "Stop here"
4. `--dry-run` stops here

## Stage 3 — Prototype

Read and substitute [prototype-agent.md](prompts/prototype-agent.md).

Spawn one **general-purpose agent per prototype** with `isolation: "worktree"`. All prototypes in single message for parallelism. Max 10 per wave.

### Constraints

- Max 200 LOC new code
- Use real code/data; stub only external APIs lacking credentials
- 1 passing smoke test required
- Branch: `proto/{project}/{feature-slug}`
- PRs marked prototype — never auto-merge
- Abandon after 30 tool calls without passing test

### Worktree Management

Each prototype gets an isolated git worktree (auto-created by `isolation: "worktree"`).
- Worktrees auto-cleanup if agent makes no changes
- If agent crashes, worktree persists — clean up manually: `git worktree list && git worktree remove <path>`
- Branch names must be unique across prototypes (use feature-slug to differentiate)

## Stage 4 — Report

```
| Project   | Feature          | ROI | Proto   | PR     | Recommendation |
|-----------|------------------|-----|---------|--------|----------------|
| alpha     | P&L dashboard    | 2.5 | WORKS   | PR #8  | Ship it        |
| pact      | Agent analytics  | 3.0 | PARTIAL | PR #12 | Iterate        |
| wellspring| Plugin system    | 1.5 | STUCK   | PR #18 | Drop           |
```

| Proto Status | ROI >= 2.0 | ROI < 2.0 |
|---|---|---|
| WORKS (test passes) | Ship it | Consider |
| PARTIAL (builds, test fails) | Iterate | Drop |
| STUCK (couldn't build) | Revisit | Drop |

"Ship it" → run `/batch` to productionize. "Iterate" → refine manually. "Drop" → close PR.

---

# SHARED

## Resume Mode

When `--resume`:
- **maintenance/push-work**: Scan for `PLAN.md` in `.batchwork/{mode}/`, parse `- [ ]` vs `- [x]`, resume Phase 2 on unchecked items. The PLAN.md can be batchwork-generated or user-authored — any `- [ ]` checklist works.
- **push-beyond**: IDEAS.md exists → skip Stage 1. PROPOSALS.md exists → skip Stage 2. Check `git branch --list 'proto/{project}/*'` → skip completed prototypes in Stage 3.

## Error Handling

| Scenario | Action |
|----------|--------|
| Invalid project path | Abort with error listing valid projects |
| Not a git repo | Warn and skip |
| Agent context exhaustion | Retry once with half the file list |
| Test fails after fix | Revert, mark `(REVERTED)`, continue |
| Prototype agent stuck (30 tool calls) | Abandon, note why in PR |
| Web search fails | Continue without competitor data |

## Agent Summary

**maintenance/push-work**: 2-5 Explore agents/project (Phase 1) → `/batch` workers (Phase 2) → main (Phases 3-5).

**push-beyond**: 1 general-purpose agent/project (Stage 1) → main (Stage 2) → 1 agent/prototype in worktree (Stage 3) → main (Stage 4).
