# Git Workflow: TelePub

## Branch Strategy

```
main          — production-ready code, auto-deploys to VPS
feature/*     — individual features (US-01 through US-05)
fix/*         — bug fixes
chore/*       — infrastructure, dependencies, docs
```

## Commit Convention

Format: `type(scope): description`

| Type | When |
|------|------|
| `feat` | New feature (maps to US-XX) |
| `fix` | Bug fix |
| `refactor` | Code restructure, no behavior change |
| `test` | Add/update tests |
| `docs` | Documentation changes |
| `chore` | Build, CI, dependencies |
| `perf` | Performance improvement |
| `security` | Security fix (use immediately, no delay) |

**Scope** = affected component: `bot`, `api`, `worker`, `db`, `payments`, `analytics`

### Examples

```bash
git commit -m "feat(bot): author registration FSM — US-01"
git commit -m "feat(payments): ЮKassa webhook handler with HMAC verification"
git commit -m "fix(worker): retry logic on channel access grant failure"
git commit -m "perf(db): add index on subscriptions.expires_at for renewal job"
git commit -m "security(api): enforce channel ownership check on all endpoints"
```

## Pre-commit Checklist

Before every commit:
```bash
# 1. Type check
mypy bot/ api/ worker/ shared/ --strict --ignore-missing-imports

# 2. Lint + format
ruff check . --fix
ruff format .

# 3. Quick tests
pytest tests/unit/ -x -q

# 4. No secrets in diff
git diff --cached | grep -iE "(token|secret|password|key)" | grep -v "# " || echo "OK"
```

## Push Discipline

- Push to `main` directly for solo dev (MVP phase)
- Never force-push to main without explicit user confirmation
- Push after each feature completion (not just at EOD)

## Release Tags

```bash
git tag -a v0.1.0 -m "MVP: author registration + subscription"
git tag -a v0.2.0 -m "v1.0: payouts + analytics"
git push origin --tags
```

## Conflict Resolution

Always rebase, never merge commits on feature branches:
```bash
git fetch origin
git rebase origin/main
# fix conflicts
git rebase --continue
```
