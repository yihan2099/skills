# Maintenance/Push-Work Research Agent Prompt

Use this prompt when spawning Explore agents for Phase 1 research.
Replace `{variables}` before passing to agent.

---

```
You are researching ~/projects/{project}/.
Mode: {mode}
Your scope: {scope}

Scan files in your scope against these categories:
{categories}

For each finding:
FINDING:
- File: {path}:{line}
- Category: {category}
- Severity: P0|P1|P2|P3
- Title: {short_description}
- Detail: {issue and evidence}
- Suggestion: {fix}

Priority: P0=data loss/security/crash, P1=likely bug, P2=code smell, P3=nice-to-have.
STOP after ~15 findings. Highest-impact first. Research only — do not edit.
```

## Scope Assignment Strategy

**Small projects (<50 source files):** Split files into 2 groups by directory.

**Medium projects (50-200 files):** Assign 3 agents by top-level directory groupings.
Example: Agent 1 = app/ + components/, Agent 2 = lib/ + utils/, Agent 3 = scripts/ + config.

**Large projects (200+ files, monorepos):** Assign agents by package/app boundary.
Example: Agent 1 = apps/web/, Agent 2 = apps/api/, Agent 3 = packages/shared/.

**Rules to prevent "Prompt is too long" errors:**
- NEVER pass individual file paths for projects with >50 files — use directory scopes
- Use relative paths from project root (e.g., `app/`, not `/home/user/projects/foo/app/`)
- Keep the category list short — use names only, not full descriptions
- Total prompt should stay under ~2000 words

## Category Shorthand

Instead of full descriptions, use this compact format in agent prompts:

**maintenance (14):** Security, Reliability, Financial Calculations, Race Conditions, Data Integrity, API Contracts, Environment & Config, Performance, Error Handling, Type Safety, Database, Auth, Dependencies, Code Quality.

**push-work (5):** In-Code TODOs, GitHub Issues, Stubbed Functions, Test Gaps, Dead Ends.

Add one-liner context only for domain-specific emphasis (e.g., "PAY SPECIAL ATTENTION to Financial Calculations — this is a trading system").
