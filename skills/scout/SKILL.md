---
name: scout
description: Weekly learning strategist. Scans your projects, market, and goals to find the highest-ROI things for YOU to learn — only skills AI can't replace.
argument-hint: "[--review] [--dry-run]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, Agent, TaskCreate, TaskUpdate, TaskList, TaskGet, AskUserQuestion, WebSearch, WebFetch, mcp__claude_ai_Notion__notion-search, mcp__claude_ai_Notion__notion-fetch, mcp__claude_ai_Notion__notion-create-pages, mcp__claude_ai_Notion__notion-update-page
---

# Scout v1.0

Weekly learning strategist. Finds the highest-ROI things for YOU to learn — not things AI handles for you.

**Invoke**: `/scout` (on-demand) or `/loop 7d /scout` (automated weekly)

## Core Filter: Human-Edge Only

**Learn these (AI can't replace):**

| Category | Examples |
|----------|----------|
| Domain Mastery | DeFi mechanics, market microstructure, geospatial systems, protocol design |
| Product & Design Sense | User research, UX intuition, product-market fit, taste |
| Business & GTM | Distribution, pricing, positioning, sales, storytelling, fundraising |
| Architecture Judgment | System design trade-offs, scaling decisions, tech bets |
| People & Leadership | Hiring, negotiation, networking, public speaking, culture |
| Regulatory & Legal | Crypto compliance, IP, contracts, licensing |
| Financial Literacy | Unit economics, fundraising mechanics, reading financials, runway |
| Mental Models | Decision-making under uncertainty, risk frameworks, first-principles |

**Skip these (AI handles well):**
Coding syntax, debugging, documentation, data analysis, research/summarization, testing, boilerplate, refactoring, code review patterns, CI/CD setup.

When a gap falls in the "skip" category, don't discard it — note it in the "AI Can Handle This" section with a one-liner on how to use AI for it instead.

## Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--review` | `false` | Review last week's plan before generating new one |
| `--dry-run` | `false` | Stop after gap analysis, print to console instead of Notion |

## Pipeline

```
Stage 1: Gather Signals    → 3 parallel agents → raw signals
Stage 2: Analyze Gaps      → Main orchestrator → gap identification
Stage 3: Score & Rank      → ROI formula → top 3-5 priorities
Stage 4: Write to Notion   → Weekly plan page
```

## Stage 1 — Gather Signals

Spawn 3 general-purpose agents in parallel (single message). Each writes to `~/projects/.scout/signals/`.

### Agent 1: Project Activity

```
Scan every git repo in ~/projects/:

1. git log --oneline --since="7 days ago" — what did the user work on?
2. git diff --stat HEAD~20..HEAD 2>/dev/null — which areas changed most?
3. Read CLAUDE.md for project context and stack
4. Look for:
   - Repeated struggles (same files touched many times)
   - Time sinks (large diffs with small net change)
   - Areas of uncertainty (TODO/FIXME with question marks)
   - New tech the user started using

Output: ~/projects/.scout/signals/projects.md
Format per project: project name, stack, this week's focus, struggle signals, growth signals
```

### Agent 2: Market & Trends

```
Based on the user's project domains, web search for:

Domains to scan (derive from ~/projects/*/CLAUDE.md):
- AI trading / quantitative finance (alpha)
- Agent economy / crypto / Base L2 / smart contracts (pact)
- Location-based AI / RAG (smalltalk)
- Personal publishing / content (yihan.app)
- General founder/indie-hacker landscape

For each domain:
1. WebSearch for developments in the last 7-14 days
2. What skills/knowledge are becoming more valuable?
3. Emerging regulations, frameworks, or paradigm shifts?
4. What are successful founders in this space learning/doing?

Output: ~/projects/.scout/signals/market.md
Format per domain: domain, key developments, skill implications, urgency
```

### Agent 3: Goals & Context

```
1. Use mcp__claude_ai_Notion__notion-fetch to read the user's 2026 planner
   (page ID: 31bbddfcf6be81468c2bfd20ca35a8c3)
2. Extract: current monthly goal, daily schedule blocks available for learning
3. Check user's stated priorities and life phase

4. If --review flag is set:
   - mcp__claude_ai_Notion__notion-search for "Scout — Week of" pages
   - Fetch the most recent one
   - Summarize what was planned vs. what happened (cross-ref with Agent 1's git data)

Output: ~/projects/.scout/signals/goals.md
Format: current phase, monthly focus, available learning time per day, review results (if any)
```

## Stage 2 — Analyze Gaps

Main orchestrator reads all three signal files. Identify gaps by cross-referencing:

1. **Struggle → Gap**: Repeated struggles in git → what knowledge would eliminate them?
2. **Market → Gap**: Industry shifts → what do you need to understand before it's urgent?
3. **Goal → Gap**: Monthly objectives → what skill is blocking the next milestone?
4. **Compound check**: Does this gap affect multiple projects? Score it higher.
5. **Human-Edge filter**: For each gap, ask: "Could I delegate this to AI instead of learning it?" If yes → move to AI-handled section. If no → keep.

## Stage 3 — Score & Rank

### Scoring: ROI = (Impact x Compound) / Time

| Factor | Scale | Description |
|--------|-------|-------------|
| Impact | 1-5 | 5 = critical blocker to revenue/launch, 1 = nice-to-know |
| Compound | 1-3 | 3 = benefits all projects, 2 = multiple, 1 = one only |
| Time | 1-5 | Hours to useful proficiency. 1 = <2h, 2 = 2-5h, 3 = 5-10h, 4 = 10-20h, 5 = 20h+ |

**Selection**: Top 3-5 by ROI. Tie-break by urgency (market-driven > self-improvement).

For each selected priority, create an **actionable learning item**:
- **Specific action** — not "learn Solidity" but "complete Damn Vulnerable DeFi challenges 1-5"
- **Time box** — fits into the user's actual schedule blocks
- **Project link** — which project benefits and how
- **Deliverable** — concrete artifact proving you learned it (a write-up, a prototype, a decision document, a conversation with someone)
- **Resource** — specific course/article/repo/person to learn from (web search for the best current resource)

## Stage 4 — Write to Notion

### Setup (first run)

1. `mcp__claude_ai_Notion__notion-search` for a page titled "Scout"
2. If not found: `mcp__claude_ai_Notion__notion-create-pages` → create "Scout" as child of the 2026 planner page

### Weekly Page

Create a new child page under "Scout":

**Title**: `Scout — Week of {YYYY-MM-DD}`

**Content structure**:

```markdown
# Scout — Week of {date}

## Review
(only if --review or previous plan exists)
| # | Item | Status | Notes |
✅ Done / 🔄 Partial (carried forward) / ❌ Skipped (dropped or deprioritized)

## Signals This Week

### Project Activity
- {project}: {one-line summary of focus + notable pattern}

### Market Moves
- {domain}: {key development + implication}

### Current Phase
{monthly goal from Notion planner}
Available learning time: {hours/week from schedule}

---

## Learning Plan

### #1 — {Title} (ROI: {score})
- **Category**: {from Core Filter table}
- **Why now**: {the signal that triggered this}
- **Action**: {specific, time-boxed task}
- **Time**: {X hours — fits in {schedule block}}
- **Applies to**: {project(s) + how}
- **Resource**: {specific URL/book/course/person}
- **Deliverable**: {what you'll produce}

### #2 — {Title} (ROI: {score})
...

### #3 — {Title} (ROI: {score})
...

---

## AI Can Handle This
(Gaps identified but filtered out — use AI instead of learning these yourself)
- {gap}: use Claude to {how}

## Backlog
(Lower-ROI items to revisit later)
- {item}: ROI {score}, revisit when {condition}
```

### Fallback

If Notion MCP is unavailable:
1. Write to `~/projects/.scout/plans/week-{YYYY-MM-DD}.md`
2. Notify user: "Notion unavailable, plan saved locally"

## --dry-run Behavior

Stop after Stage 3. Print the scored plan to console. Don't write to Notion.

## Review Mode (--review)

When `--review` is passed (or when a previous week's Scout page is detected):

1. Agent 3 fetches last week's Scout page from Notion
2. Cross-reference each planned item with Agent 1's git activity:
   - Did related project work happen?
   - Was the deliverable produced?
3. Score: ✅ Done / 🔄 Partial / ❌ Skipped
4. Carry forward incomplete items with ROI >= 2.0
5. Include review section at top of new plan

## Cleanup

After Stage 4 completes successfully, delete `~/projects/.scout/signals/` (intermediate files). Keep `~/projects/.scout/plans/` as local backup.

## Error Handling

| Scenario | Action |
|----------|--------|
| Notion MCP unavailable | Fallback to local file, notify user |
| No git activity this week | Skip project signals, weight market + goals higher |
| WebSearch fails | Continue without market signals, note the gap |
| No previous Scout page for --review | Skip review section, generate fresh |
| Agent context exhaustion | Retry with fewer projects per agent |
