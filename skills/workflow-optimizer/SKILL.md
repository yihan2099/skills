---
name: optimize
description: Closed-loop workflow optimization. Runs a workflow N times, categorizes failures, applies fixes, re-measures until target success rate or max iterations.
argument-hint: "<workflow-path> [--runs 5] [--target-rate 0.8] [--max-iterations 5] [--baseline-only]"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
---

# Optimize — Closed-Loop Workflow Improvement

All logic is inline — this skill does NOT invoke other skills.

## Input Parsing

1. **workflow-path** — path to `workflow.md`
2. **--runs N** (default: 5) — runs per iteration
3. **--target-rate N** (default: 0.8) — target success rate (0.0–1.0)
4. **--max-iterations N** (default: 5) — max fix-and-retest cycles
5. **--stagnation-limit N** (default: 3) — stop after N iterations with no improvement
6. **--baseline-only** — measure once and stop

## The Loop

```
LOAD workflow.md
  ↓
MEASURE (N runs)
  ↓
success_rate >= target? ──YES──→ DONE
  │ NO
  ↓
CATEGORIZE failures
  ↓
all unfixable? ──YES──→ STOP (explain why)
  │ NO
  ↓
FIX (apply fixer.md for dominant category)
  ↓
REBUILD (if workflow requires it)
  ↓
REMEASURE (N runs)
  ↓
COMPARE (before vs after)
  ↓
SAVE iteration
  ↓
stagnant for K iterations? ──YES──→ STOP
  │ NO
  ↓
LOOP back to success_rate check
```

---

## Phase 1: Load Workflow

Read the `workflow.md` file. Extract these required fields:

| Field | Description |
|-------|-------------|
| **Run Command** | Shell command, API call, or agent prompt template |
| **Success Criteria** | How to check if a run passed (exit code, output pattern, URL pattern) |
| **Fixtures** | Test inputs substituted into the run command |
| **Constraints** | Timeout (ms), max turns (agent workflows) |

Optional fields:
- **Rebuild Command** — command to run after applying fixes (e.g., Docker rebuild)
- **Setup Command** — one-time setup before first run

If a setup command exists, run it now.

Also check for `fixer.md` in the same directory as `workflow.md`. This is where domain-specific fix instructions live.

---

## Phase 2: Measure

Run the workflow N times and collect structured results.

### Executing Each Run

For each run `i` in `1..N`:
1. Pick fixture via round-robin: `fixtures[i % fixtures.length]`
2. Substitute fixture values into the run command template
3. Execute and capture: exit code, stdout, stderr, wall-clock duration
4. Check success criteria against the output
5. Record the result

### Run Result Format

```markdown
## Run {i}
- **Fixture**: {fixture_id}
- **Success**: PASS / FAIL
- **Duration**: {ms}ms
- **Error**: {error message or "none"}
- **Output** (truncated to 500 chars): {stdout}
```

### Timeout Handling

If a run exceeds the workflow's timeout constraint, kill it and record `error: "TIMEOUT"`.

### Parallel Runs (Optional)

For independent runs (no shared state between runs), use the `Agent` tool to run multiple fixtures concurrently:

```
Spawn up to 3 agents, each running a subset of the N runs.
Agent 1: runs 1, 4, 7...
Agent 2: runs 2, 5, 8...
Agent 3: runs 3, 6, 9...
```

Each agent writes results to `.workflow-optimizer/{workflow-id}/runs/run-{i}.md`.

### Aggregate Metrics

After all runs complete, compute:

| Metric | Formula |
|--------|---------|
| Success rate | pass_count / N |
| Avg duration | mean(duration_ms) |
| Failure distribution | count per error category |

Write aggregate to `.workflow-optimizer/{workflow-id}/baseline.md` (first iteration) or compare against it (subsequent iterations).

If `--baseline-only`, print the report and **STOP**.

---

## Phase 3: Check Target

If `success_rate >= target_rate` → print summary and **STOP**. The workflow already meets the goal.

---

## Phase 4: Categorize Failures

For each failed run, match the error text against these patterns (first match wins):

| Category | Patterns | Fixable? |
|----------|----------|----------|
| AUTH | logged out, sign in, expired session, unauthorized | No — needs manual re-login |
| RATE_LIMIT | 429, rate limit, too many requests | No — wait required |
| TIMEOUT | timeout, timed out, deadline, turn limit exceeded | Yes |
| UI_CHANGE | selector not found, element not found, no such element | Yes |
| TIMING | not ready, loading, spinner, pending | Yes |
| CONTENT | text empty, failed to type, paste fail | Yes |
| BROWSER | browser crash, chromium, target closed, page closed | Partial |
| UNKNOWN | (no pattern matched) | Review needed |

Output a failure distribution:

```
Failures (3/5 runs):
  TIMEOUT    2  (fixable)
  AUTH       1  (not fixable)
Fixable: 2/3 (67%)
```

If ALL failures are unfixable (AUTH, RATE_LIMIT) → **STOP** with explanation.

---

## Phase 5: Fix

### With fixer.md (Recommended)

If the workflow directory contains `fixer.md`, read it. The fixer maps each failure category to specific fix instructions for this workflow.

Example fixer.md:
```markdown
# Fixer: my-workflow

## TIMEOUT
- Chain sequential commands with &&
- Remove unnecessary intermediate steps

## TIMING
- Add sleep 2 before element interactions
- Add retry loop for NOT_READY elements

## UI_CHANGE
- Inspect current page structure
- Update selectors in the run command/prompt
```

Apply fixes for the **dominant failure category** (highest count). Make surgical changes — don't rewrite everything.

### Without fixer.md (Auto-Fix)

If no fixer.md exists, apply generic fixes based on category:

| Category | Generic Fix |
|----------|------------|
| TIMEOUT | Reduce steps, chain commands, increase timeout |
| TIMING | Add waits/retries before interactions |
| UI_CHANGE | Read current page/API and update selectors |
| CONTENT | Switch input method, add validation |

After fixing, update the workflow's run command or prompt accordingly.

---

## Phase 6: Rebuild

If the workflow defines a **Rebuild Command** in `workflow.md`, run it now:

```bash
# Example: Docker-based workflow
docker stop my-container && docker rm my-container
docker build -t my-image .
docker run -d --name my-container ...
```

Skip this phase if no rebuild command is defined.

---

## Phase 7: Remeasure

Run Phase 2 again with the updated workflow. Same N runs, same fixtures.

---

## Phase 8: Compare

Compute deltas between previous and current metrics:

| Metric | Before | After | Delta | Better if |
|--------|--------|-------|-------|-----------|
| Success rate | | | | Higher |
| Avg duration | | | | Lower |

**Improved** = success rate increased, OR same rate with lower duration.
**Regressed** = success rate decreased.
**Stagnant** = no meaningful change.

```
COMPARISON:
  Success Rate   60.0% → 80.0%  (+20.0%)
  Avg Duration   185s  → 120s   (-65s)
  Verdict: IMPROVED
```

---

## Phase 9: Save Iteration

Write to `.workflow-optimizer/{workflow-id}/iterations/iter-{N}.md`:

```markdown
# Iteration {N} — {date}

## Changes Made
- {description of fixes applied}

## Results
| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| Success Rate | 60% | 80% | +20% |
| Avg Duration | 185s | 120s | -65s |

## Failure Distribution
| Category | Before | After | Delta |
|----------|--------|-------|-------|
| TIMEOUT | 3 | 1 | -2 |
```

Update `.workflow-optimizer/{workflow-id}/baseline.md` with the new metrics.

---

## Phase 10: Stagnation Check

Track consecutive iterations with no improvement (success rate didn't increase).

If `stagnation_count >= stagnation_limit` → **STOP**.
- Report what was achieved vs target
- List remaining failure categories

Otherwise, loop back to Phase 3.

---

## Storage Layout

```
.workflow-optimizer/
  {workflow-id}/
    baseline.md              # Latest aggregate metrics
    iterations/
      iter-1.md              # Per-iteration changes + results
      iter-2.md
    failure-catalog.md       # Accumulated failure patterns + fixes
```

Append new failure patterns to `failure-catalog.md` after each iteration:

```markdown
## TIMEOUT
- **Pattern**: Turn limit exceeded at 30 turns
- **Root cause**: Too many sequential commands
- **Fix applied**: Chained commands with &&
- **Date**: {date}
```

---

## Final Report

```
============================================================
OPTIMIZATION SUMMARY
============================================================
  Converged    : YES/NO
  Iterations   : {N}
  Total Runs   : {N * runs_per_iteration}
  Initial Rate : XX.X%
  Final Rate   : XX.X%
  Improvement  : +XX.X%
============================================================
```

If converged, suggest committing the changes. If not, list remaining failure categories.
