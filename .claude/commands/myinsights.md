# /myinsights — Capture Development Insights

Capture and persist non-obvious learnings from the current development session. These become project memory for future sessions.

## Usage

```
/myinsights
/myinsights --session="payments integration"
/myinsights --type=gotcha          # only capture gotchas
/myinsights --type=decision        # only capture architectural decisions
```

## When to use

Run at the end of a development session, after fixing a tricky bug, or when you discover something non-obvious about:
- ЮKassa API behavior
- aiogram edge cases
- SQLAlchemy async gotchas
- Telegram API quirks
- Performance bottlenecks

## What gets captured

### Categories

| Type | Description | Example |
|------|-------------|---------|
| `gotcha` | Unexpected behavior that burned time | "ban+unban trick for removing channel members" |
| `decision` | Architectural choice with rationale | "Redis for FSM instead of DB to avoid async issues" |
| `pattern` | Reusable code pattern discovered | "selectinload pattern for all subscription queries" |
| `integration` | Third-party API quirk | "ЮKassa idempotency_key must be NEW UUID per retry" |
| `performance` | Optimization discovered | "Index on expires_at WHERE status='active' cuts renewal job 10x" |

## Process

1. Review current session's changes:
   ```bash
   git diff HEAD~5..HEAD --stat
   git log --oneline -5
   ```

2. Identify non-obvious learnings (skip things obvious from reading the code)

3. For each insight, write to `docs/insights/YYYY-MM-DD-topic.md`:

```markdown
---
date: YYYY-MM-DD
type: gotcha|decision|pattern|integration|performance
component: bot|api|worker|db|payments|telegram
severity: critical|important|minor
---

## [Short title]

**What happened:** [The situation]

**What was non-obvious:** [The surprise]

**Solution/Pattern:**
[Code snippet or description]

**Why this matters:** [Impact on the project]
```

4. If insight is critical → also add to `CLAUDE.md` under "Development Insights"

5. Commit insights:
   ```bash
   git add docs/insights/
   git commit -m "docs: development insights — [topic]"
   ```

## Auto-capture from hooks

The `.claude/settings.json` hook captures insights automatically after each session. Manual `/myinsights` is for explicit, structured capture of important learnings.

## Review insights

```bash
ls docs/insights/
cat docs/insights/*.md | grep "## " | sort   # list all insight titles
grep "type: critical" docs/insights/*.md      # show only critical insights
```
