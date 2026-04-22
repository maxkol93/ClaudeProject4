# /feature — Full Feature Development Lifecycle

End-to-end feature development from spec to tested, committed code.

## Usage

```
/feature registration         # Implement US-01: Author Registration
/feature subscription         # Implement US-02: Subscription Purchase
/feature lifecycle            # Implement US-03: Access Management
/feature payouts              # Implement US-04: Author Payouts
/feature analytics            # Implement US-05: Analytics Dashboard
/feature [custom-name]        # Any feature from docs/features/
```

## Pipeline

```
PLAN → SPEC → IMPLEMENT → TEST → REVIEW → COMMIT
```

### Step 1: PLAN (2-3 min)

Read the relevant user story from `docs/PRD.md` and algorithm from `docs/Pseudocode.md`.

Output a brief implementation plan:
```
## Feature: [name]
**Story:** US-XX
**Estimated files:** N
**Dependencies:** [list]
**Implementation order:**
1. [data model / migration]
2. [service / business logic]
3. [handler / API endpoint]
4. [Celery task if async]
5. [tests]
```

Confirm plan with user before proceeding.

### Step 2: SPEC (auto, no confirmation needed)

Create `docs/features/[name].md` with:
- Feature scope (in/out of this PR)
- Algorithm from Pseudocode.md
- Edge cases from Refinement.md
- Test scenarios from test-scenarios.md

### Step 3: IMPLEMENT

Follow `docs/Architecture.md` patterns strictly.

**File creation order:**
1. `shared/models/[entity].py` — SQLAlchemy model + Alembic migration
2. `shared/services/[name].py` — business logic, pure functions
3. `bot/handlers/[name].py` OR `api/routers/[name].py` — entry point
4. `worker/tasks/[name].py` — async Celery tasks (if needed)

**Coding rules (from .claude/rules/coding-style.md):**
- Python 3.12+, type hints everywhere
- All I/O async (never block event loop)
- SQLAlchemy: always `selectinload`, never lazy loading
- aiogram: FSM via `StatesGroup`, state in Redis
- Celery: `bind=True, max_retries=5, default_retry_delay=60`
- Payments: HMAC verification FIRST, then business logic

**Security rules (from .claude/rules/security.md):**
- HMAC-SHA256 verify webhooks before any DB access
- JWT + channel ownership check on every API endpoint
- No raw SQL with user input

### Step 4: TEST

Generate tests following `docs/Refinement.md` strategy:

```bash
# Unit tests
pytest tests/unit/test_[name].py -v

# Integration tests (requires running PostgreSQL + Redis)
pytest tests/integration/test_[name].py -v --tb=short

# Type checking
mypy [module]/ --strict

# Linting
ruff check [module]/ --fix
```

Minimum coverage target: 80% for new code.

### Step 5: REVIEW

Self-review checklist before commit:
- [ ] Type hints on all functions
- [ ] No blocking calls in async context
- [ ] HMAC verification present (if webhook handler)
- [ ] Idempotency key used (if payment operation)
- [ ] Celery task has max_retries (if async task)
- [ ] Index exists for queried column
- [ ] No hardcoded secrets

### Step 6: COMMIT

```bash
git add -p    # stage changes interactively
git commit -m "feat(US-XX): [description]

- [key change 1]
- [key change 2]

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"

git push origin main
```

## Feature-to-file mapping

| Feature | Key files |
|---------|-----------|
| registration | `bot/handlers/registration.py`, `shared/models/author.py`, `shared/models/channel.py` |
| subscription | `bot/handlers/subscription.py`, `worker/tasks/access.py`, `api/routers/webhooks.py` |
| lifecycle | `worker/tasks/lifecycle.py`, `worker/beat/schedules.py` |
| payouts | `worker/tasks/payouts.py`, `api/routers/payouts.py` |
| analytics | `api/routers/analytics.py`, `shared/services/analytics.py` |
