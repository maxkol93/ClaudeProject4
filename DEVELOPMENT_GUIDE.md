# TelePub — Development Guide

Telegram-native channel monetization platform for the Russian market.

---

## Quick start (new developer)

```bash
# 1. Clone
git clone https://github.com/maxkol93/ClaudeProject4 telepub
cd telepub

# 2. Create secrets for local dev
mkdir -p secrets/
echo "your_telegram_bot_token" > secrets/telegram_bot_token.txt
echo "your_yukassa_shop_id" > secrets/yukassa_shop_id.txt
echo "your_yukassa_secret" > secrets/yukassa_secret_key.txt
echo "$(python3 -c 'import secrets; print(secrets.token_hex(32))')" > secrets/jwt_secret.txt
echo "devpassword123" > secrets/postgres_password.txt
echo "devredispass123" > secrets/redis_password.txt
echo "webhook_secret_$(date +%s)" > secrets/telegram_webhook_secret.txt

# 3. Start infra
docker compose up -d postgres redis

# 4. Run migrations
docker compose run --rm api alembic upgrade head

# 5. Start services in dev mode
docker compose up bot api worker
```

---

## Project structure

```
telepub/
├── bot/                  # Telegram Bot (aiogram 3.x)
│   ├── handlers/         # Message + callback handlers
│   ├── keyboards/        # Inline keyboards
│   ├── middlewares/      # Auth, rate limiting
│   └── states.py         # FSM StatesGroups
│
├── api/                  # REST API (FastAPI)
│   ├── routers/          # Endpoints: auth, channels, analytics, webhooks, payouts
│   ├── dependencies.py   # JWT auth, session injection
│   └── schemas/          # Pydantic request/response models
│
├── worker/               # Async tasks (Celery)
│   ├── tasks/            # access.py, lifecycle.py, payouts.py, notifications.py
│   ├── beat/             # Scheduled jobs
│   └── celery_app.py     # Celery config
│
├── shared/               # Shared code
│   ├── models/           # SQLAlchemy models (Author, Channel, Subscription, Payment, Payout)
│   ├── services/         # Business logic (payments, analytics, subscriptions)
│   ├── database.py       # Async engine + session factory
│   └── config.py         # Settings from Docker secrets
│
├── alembic/              # DB migrations
│   └── versions/         # Migration files
│
├── tests/
│   ├── unit/             # Pure function tests (no external deps)
│   ├── integration/      # Real DB + Redis (testcontainers)
│   └── conftest.py       # Shared fixtures
│
├── nginx/                # Nginx config
├── monitoring/           # Prometheus + Grafana configs
├── docs/                 # SPARC documentation
│   ├── PRD.md            # Product requirements + user stories
│   ├── Architecture.md   # System architecture
│   ├── Pseudocode.md     # Core algorithms
│   ├── Refinement.md     # Edge cases + risks
│   └── test-scenarios.md # BDD scenarios
└── docker-compose.yml    # All services
```

---

## Feature development workflow

### 1. Pick a feature

Check `CLAUDE.md` → Feature Roadmap for what's next.

```
/start             # Load project context, see recommended next feature
/plan [feature]    # Create implementation plan
```

### 2. Understand the requirements

```bash
cat docs/PRD.md            # User story + acceptance criteria
cat docs/Pseudocode.md     # Algorithm for the feature
cat docs/Refinement.md     # Edge cases
grep "Feature:" docs/test-scenarios.md  # BDD scenarios
```

### 3. Plan before coding

Always confirm a plan before writing code. Use `/plan [feature]` to generate it.

### 4. Implement in order

```
1. shared/models/[entity].py     + alembic migration
2. shared/services/[name].py     (business logic, pure functions)
3. bot/handlers/ OR api/routers/ (entry point)
4. worker/tasks/[name].py        (async Celery task)
5. tests/unit/ + tests/integration/
```

### 5. Validate

```bash
mypy bot/ api/ worker/ shared/ --strict --ignore-missing-imports
ruff check . --fix
pytest tests/unit/ -v -x
pytest tests/integration/ -v
```

### 6. Commit and push

```bash
git add [specific files]
git commit -m "feat(bot): author registration FSM — US-01

- Add RegistrationStates FSM
- Implement /register command handler  
- Add bot admin verification step

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"

git push origin main
```

---

## MVP feature checklist

| Feature | Story | Status |
|---------|-------|--------|
| Author Registration FSM | US-01 | [ ] Planned |
| Subscription Payment Flow | US-02 | [ ] Planned |
| Auto Access Management | US-03 | [ ] Planned |
| Author Payouts | US-04 | [ ] Planned |
| Analytics Dashboard | US-05 | [ ] Planned |

---

## Key domain rules

### Commission
- 0% platform fee on first 10,000 RUB/month per author
- 5% platform fee above threshold
- `net_to_author = amount - platform_fee - yukassa_fee`

### Access management
- Grant: `create_chat_invite_link(member_limit=1)` → single-use link → DM to subscriber
- Revoke: `ban_chat_member` then immediately `unban_chat_member` (removes without blocking re-subscribe)

### Payments
- HMAC-SHA256 verify EVERY webhook BEFORE any business logic
- `idempotency_key` = new `uuid.uuid4()` per payment attempt, never reuse
- Save `payment_method_id` from succeeded webhook for rebilling
- Duplicate webhook → check existing status → skip if already processed

### Renewals
- Daily Celery Beat job at 09:00 MSK
- Query subscriptions expiring in ≤3 days
- Attempt rebilling via saved payment_method_id
- On failure: grace period (24h) → expired + revoke access

### Payouts
- Minimum balance: 1,000 RUB
- Daily auto-payout at 10:00 MSK
- Manual payout via `/payout` bot command
- ЮKassa payout limit: 600,000 RUB per transaction (split if needed)

---

## Critical gotchas

1. **ban+unban pattern** — never just `ban_chat_member` for revocation (permanent block)
2. **Redis FSM** — MemoryStorage loses state on restart; use RedisStorage in production
3. **selectinload** — never use lazy loading in async SQLAlchemy context
4. **idempotency_key uniqueness** — never reuse subscription_id as idempotency_key
5. **HMAC first** — webhook signature verification must be the VERY FIRST operation

---

## Available Claude commands

| Command | Description |
|---------|-------------|
| `/start` | Load project context, see recommended next feature |
| `/feature [name]` | Full feature lifecycle: plan → build → test → commit |
| `/plan [feature]` | Implementation plan only (no code) |
| `/test [scope]` | Run or generate tests |
| `/deploy` | Deploy to VPS |
| `/myinsights` | Capture development insights |

## Available agents

| Agent | Use when |
|-------|---------|
| `planner` | Before implementing any feature |
| `code-reviewer` | After implementing, before committing |
| `architect` | Infrastructure or design decisions |

---

## External resources

- [aiogram 3.x docs](https://docs.aiogram.dev/en/stable/)
- [ЮKassa Python SDK](https://github.com/yoomoney/yookassa-sdk-python)
- [FastAPI docs](https://fastapi.tiangolo.com/)
- [SQLAlchemy 2.0 async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Celery docs](https://docs.celeryq.dev/en/stable/)
