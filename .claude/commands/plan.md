# /plan — Implementation Planning

Create a detailed implementation plan for a feature or subsystem.

## Usage

```
/plan registration         # Plan US-01: Author Registration FSM
/plan payments             # Plan ЮKassa integration
/plan renewals             # Plan daily renewal job
/plan analytics            # Plan analytics aggregation
/plan [topic]              # Any technical area
```

## Process

1. Read relevant docs (PRD, Pseudocode, Architecture)
2. Identify all files to create/modify
3. Map dependencies between files
4. Output ordered plan with estimates

## Output format

```markdown
## Plan: [feature]

### Files to create
| Order | File | Purpose | ~Lines |
|-------|------|---------|--------|
| 1 | shared/models/author.py | SQLAlchemy model | ~50 |
| 2 | alembic/versions/001_author.py | Migration | ~30 |
| 3 | shared/services/auth.py | Registration logic | ~80 |
| 4 | bot/handlers/registration.py | FSM handler | ~120 |
| 5 | tests/unit/test_registration.py | Unit tests | ~100 |

### Files to modify
| File | Change |
|------|--------|
| shared/models/__init__.py | Import Author model |

### Dependency order
[description of why this order matters]

### Edge cases pre-identified
[from docs/Refinement.md]

### Estimated session time
[rough estimate]
```

## Confirm before implementing

Always output the plan and wait for explicit "OK" before writing any code.
The plan is an alignment artifact, not a prelude to immediate action.
