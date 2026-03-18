# Push-Beyond Prototype Agent Prompt

Use this prompt when spawning builder agents for Stage 3 prototyping.
Spawn with `isolation: "worktree"` for git isolation.
Replace `{variables}` before passing to agent.

---

```
You are building a prototype for {project}.

Feature: {title}
Description: {from PROPOSALS.md prototype spec}
Entry point: {where to wire it in}
Smoke test: {how to verify}

RULES:
1. Create branch: proto/{project}/{feature-slug}
2. Build the MINIMUM working version — target < 200 LOC of new code
3. Reuse existing code, data, and infrastructure wherever possible
4. Wire it into the real codebase (import real modules, use real data)
5. For external APIs requiring credentials you don't have, stub the HTTP call
   with a placeholder and test the wiring — do NOT mock internal code
6. Write exactly 1 smoke test that proves the happy path works
7. Run the smoke test — it must pass
8. Commit with message: "proto: {feature title}"
9. Create PR with this body format:

   ## Prototype: {Title}
   **ROI**: {score} (Impact {I} / Effort {E})

   ### What it does
   {One paragraph}

   ### Demo
   {Output, screenshot description, or test result}

   ### What's missing for production
   - {item 1}
   - {item 2}

   ### Effort estimate for production
   {Rough estimate}

   This is a prototype branch — do not merge directly.

CONSTRAINTS:
- Do NOT refactor existing code
- Do NOT add error handling beyond what's needed for the demo
- Do NOT write documentation
- If you've made more than 30 tool calls without a passing test, abandon
  and note why in the PR description
```
