---
name: feature-navigator
description: TelePub feature navigator — determines the optimal next feature to implement based on PRD priority, dependencies, and current project state. Use when unsure what to work on next.
version: "1.0"
maturity: production
---

# TelePub — Feature Navigator

Recommends the next feature to implement based on PRD priority and dependencies.

## MVP feature dependency graph

```
US-01 (Registration)
  └── US-02 (Subscription) — requires Author + Channel models
        └── US-03 (Lifecycle) — requires Subscription + Payment models
              └── US-04 (Payouts) — requires Payout model + balance tracking
                    └── US-05 (Analytics) — requires all above data
```

## Recommendation logic

1. Check what's already built:
   ```bash
   find bot/handlers/ api/routers/ worker/tasks/ -name "*.py" 2>/dev/null | sort
   git log --oneline | grep "feat("
   ```

2. Identify lowest unimplemented story with no unmet dependencies

3. Output recommendation:
   ```markdown
   ## Recommended next: US-XX — [title]
   
   **Why:** [dependency satisfied, highest priority unimplemented]
   
   **Start with:** /feature [name]
   
   **Key files:**
   - [file 1]
   - [file 2]
   
   **Estimated complexity:** S/M/L
   ```

## Feature metadata

| Story | Command | Complexity | Key algorithm |
|-------|---------|------------|---------------|
| US-01 | `/feature registration` | M | Registration FSM (5 states) |
| US-02 | `/feature subscription` | L | Payment flow + webhook handler |
| US-03 | `/feature lifecycle` | L | Daily renewal + grace period |
| US-04 | `/feature payouts` | M | ЮKassa payout + balance tracking |
| US-05 | `/feature analytics` | M | Aggregation + Redis caching |

## First-time recommendation

If nothing is built yet, always start with US-01 (Author Registration):
- No dependencies
- Creates the foundational Author + Channel models
- Enables all subsequent features
- Most isolated (bot-only, no payments)

Command: `/feature registration`
