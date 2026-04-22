# Insights Capture Protocol: TelePub

## What counts as an insight

Only capture information that would NOT be obvious to a developer reading the code cold.

**Capture:**
- Unexpected API behavior (Telegram, ЮKassa)
- Race conditions and how they were resolved
- Why a "simpler" approach was tried and failed
- Performance discoveries with specific metrics
- Architectural decisions that might look wrong but are intentional

**Skip:**
- Things already in `docs/Architecture.md` or `docs/Refinement.md`
- Standard Python/FastAPI/aiogram usage
- Anything obvious from the code itself

## Insight file format

Location: `docs/insights/YYYY-MM-DD-topic.md`

```markdown
---
date: YYYY-MM-DD
type: gotcha|decision|pattern|integration|performance
component: bot|api|worker|db|payments|telegram
severity: critical|important|minor
---

## [Short title — max 60 chars]

**What happened:** [Context in 1-2 sentences]

**What was non-obvious:** [The surprise]

**Solution:**
[Code snippet or clear description — keep it short]

**Impact:** [Why this matters for TelePub specifically]
```

## Critical insights → CLAUDE.md

If severity is `critical`, also add a one-liner to `CLAUDE.md` under the "Development Insights" section.

## Trigger for capture

After any session that involved:
- Debugging a payment flow
- Fixing an aiogram FSM issue
- Discovering a Telegram API limit
- Resolving a race condition
- Optimizing a slow query

Run: `/myinsights`

## Examples from known gotchas

```markdown
---
date: 2026-04-22
type: gotcha
component: telegram
severity: critical
---

## ban+unban = remove without block

To remove a subscriber from a private channel, you must:
1. `ban_chat_member(chat_id, user_id)` — kicks the user
2. Immediately `unban_chat_member(chat_id, user_id)` — removes from ban list

Only ban = they can never rejoin. ban+unban = clean removal, re-subscribe possible.

**Impact:** All revoke_channel_access implementations must use this pattern.
```
