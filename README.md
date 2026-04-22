# TelePub

Telegram-native channel monetization platform for the Russian market.

Authors connect their Telegram channels, set subscription prices, and accept recurring payments — all through a single Telegram bot. Subscribers pay via ЮKassa (Russian payment provider) and get automatic access to private channels.

**Competitors defeated:** Boosty (10-20% fee, complex UX), Tribute (5% fee, limited features)  
**Differentiation:** 0→5% freemium commission, 24h payouts, Telegram-first UX

---

## Architecture

Distributed Monolith in Monorepo. Single VPS, Docker Compose, Python everywhere.

```
Telegram → Nginx → Bot (aiogram 3.x) ─┐
                 → API (FastAPI)       ├─→ PostgreSQL 16
ЮKassa   → Nginx → API               ─┘   Redis 7
                 → Worker (Celery)    ─→  ЮKassa API
```

## Tech stack

| Layer | Technology |
|-------|------------|
| Bot | Python 3.12, aiogram 3.x |
| API | FastAPI 0.115+, uvicorn |
| Tasks | Celery 5.x, Redis broker |
| DB | PostgreSQL 16, SQLAlchemy 2.0 async, Alembic |
| Cache | Redis 7 |
| Payments | ЮKassa Python SDK |
| Frontend | Next.js 14 (web dashboard + TWA) |
| Infrastructure | Docker Compose, VPS AdminVPS/HostKey |
| AI Analytics | Claude claude-sonnet-4-6 via MCP |

## Quick start

```bash
# Setup secrets (local dev)
mkdir secrets/
echo "your_bot_token" > secrets/telegram_bot_token.txt
# ... (see DEVELOPMENT_GUIDE.md for full list)

# Start
docker compose up -d postgres redis
docker compose run --rm api alembic upgrade head
docker compose up bot api worker
```

## MVP features

| Story | Feature |
|-------|---------|
| US-01 | Author registers channel via bot FSM (5 minutes) |
| US-02 | Subscriber pays via ЮKassa, gets channel access in 30s |
| US-03 | Auto-renewals, grace periods, access revocation |
| US-04 | Author receives payouts within 24h |
| US-05 | Analytics dashboard (MRR, subscribers, churn) |

## Documentation

All product and technical documentation in `docs/`:

- [`docs/PRD.md`](docs/PRD.md) — Product requirements and user stories
- [`docs/Architecture.md`](docs/Architecture.md) — System architecture
- [`docs/Pseudocode.md`](docs/Pseudocode.md) — Core algorithms
- [`docs/Refinement.md`](docs/Refinement.md) — Edge cases and risks
- [`docs/test-scenarios.md`](docs/test-scenarios.md) — 36 BDD scenarios
- [`DEVELOPMENT_GUIDE.md`](DEVELOPMENT_GUIDE.md) — Developer onboarding

## Development

```bash
/start             # Load project context
/feature [name]    # Implement a feature end-to-end
/test              # Run tests
/deploy            # Deploy to VPS
```

## Commission model

- 0% fee: author monthly revenue ≤ 10,000 RUB
- 5% fee: author monthly revenue > 10,000 RUB
- Payouts: daily auto at balance ≥ 1,000 RUB, target 24h delivery
