# Feature Lifecycle: TelePub

## 6-step lifecycle for every feature

```
1. PLAN    → align on scope
2. SPEC    → document what to build
3. BUILD   → implement
4. TEST    → verify
5. REVIEW  → quality gate
6. SHIP    → commit + push
```

## Step 1: PLAN (always first, always explicit)

Before writing code:
1. Read the user story in `docs/PRD.md`
2. Read the algorithm in `docs/Pseudocode.md`
3. Read edge cases in `docs/Refinement.md`
4. Output a file list (what will be created/modified)
5. Get explicit "OK" from user before writing code

Never start implementing without a confirmed plan.

## Step 2: SPEC

Create `docs/features/[name].md`:
- Story scope (what's IN this feature, what's OUT)
- Algorithm steps from Pseudocode.md
- Edge cases to handle
- Test scenarios to cover

## Step 3: BUILD

**Implementation order (always follow this sequence):**

```
1. Database model (shared/models/)
2. Alembic migration
3. Service layer (shared/services/)
4. Bot handler OR API router
5. Celery task (if async work needed)
6. Notification logic
```

**Per-file checklist:**
- Type hints on every function signature
- `async def` for every DB or network call
- `selectinload`/`joinedload` for SQLAlchemy relations
- No `requests`, no `time.sleep` in async context

## Step 4: TEST

Required test coverage per feature:

| Test type | Required | Tool |
|-----------|----------|------|
| Unit — happy path | YES | pytest |
| Unit — error cases | YES | pytest |
| Integration — DB | YES | pytest + testcontainers |
| Type check | YES | mypy --strict |
| Lint | YES | ruff |
| Manual smoke test | YES | actual Telegram interaction |

```bash
# Full test run for a feature
pytest tests/unit/test_[name].py tests/integration/test_[name].py -v
mypy [module]/ --strict
ruff check [module]/
```

## Step 5: REVIEW

Self-review checklist (run before every commit):

**Security:**
- [ ] Webhook handler: HMAC verification is the FIRST thing executed
- [ ] API endpoint: JWT verified + channel ownership checked
- [ ] No secrets in code (use `os.getenv` or Docker secrets path)
- [ ] No user input in raw SQL

**Payments:**
- [ ] idempotency_key is a NEW UUID (not subscription_id)
- [ ] Webhook handler handles duplicate delivery (check payment status)
- [ ] payment_method_id saved from succeeded webhook for rebilling

**Telegram:**
- [ ] Rate limiting considered for bulk operations
- [ ] Invite links are `create_chat_invite_link(member_limit=1)` (single-use)
- [ ] Access revoke uses ban+unban pattern (not just ban)

**Reliability:**
- [ ] Celery task has `max_retries=5, default_retry_delay=60`
- [ ] Exponential backoff: `countdown=2 ** self.request.retries`
- [ ] Failure case logs enough context to debug

## Step 6: SHIP

```bash
git add [specific files]     # never git add -A
git commit -m "feat(component): description — US-XX

- specific change 1
- specific change 2

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"

git push origin main
```

## Feature status tracking

Update `CLAUDE.md` feature roadmap after shipping:
- Move feature from "next" to "in_progress" when starting
- Move to "complete" when pushed

## Post-ship

Run `/myinsights` to capture any non-obvious learnings from implementation.
