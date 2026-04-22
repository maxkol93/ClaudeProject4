# TelePub — Claude Code Context

## Overview

TelePub — Telegram-native платформа монетизации каналов для авторов РФ/СНГ. Авторы принимают платные подписки, управляют доступом и анализируют аудиторию через Telegram-бот. Встроенная рекламная биржа (v2.0) создаёт дополнительный revenue stream.

**Целевая аудитория:** Авторы Telegram-каналов 1K–100K подписчиков (RU/CIS)
**Бизнес-модель:** 0% комиссии до 10K руб/мес → 5% после + рекламная биржа 20%

## Problem & Solution

Авторы Telegram-каналов РФ/СНГ не имеют удобного инструмента монетизации: Boosty берёт 10–20%, Tribute примитивен, Telegram Stars — 30% Apple cut. TelePub решает это через Telegram-native бот с ЮKassa-платежами, выплатами за 24 часа и AI-аналитикой.

## Architecture

```
Distributed Monolith (Monorepo) — Docker Compose на VPS AdminVPS/HostKey

telepub/
├── bot/          ← aiogram 3.x (Telegram Bot API, FSM)
├── api/          ← FastAPI (REST API, Web Dashboard, TWA)
├── worker/       ← Celery (платежи, доступ к каналам, выплаты)
├── beat/         ← Celery Beat (ежедневный renewal, payouts)
├── frontend/     ← Next.js 14 (Web Dashboard, TWA)
└── shared/       ← Общие модели, утилиты
```

Инфраструктура: Nginx → Bot:8001 + API:8000 | PostgreSQL 16 + Redis 7 + MinIO

## Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Bot | aiogram 3.x + aiogram-dialog | FSM в Redis |
| API | FastAPI + uvicorn | Async, OpenAPI |
| ORM | SQLAlchemy 2.0 async + Alembic | PostgreSQL |
| Tasks | Celery 5.x + Redis broker | 3 очереди |
| Frontend | Next.js 14 (TypeScript) | SSR + TWA |
| DB | PostgreSQL 16 | Основное хранилище |
| Cache | Redis 7 | FSM, rate limit, queue |
| Files | MinIO | S3-compatible |
| Payments | ЮKassa Python SDK | Primary payment |
| Deploy | Docker Compose | VPS AdminVPS/HostKey |
| Proxy | Nginx 1.25 | SSL + rate limit |

## Key Algorithms

```python
# Расчёт комиссии (Pseudocode.md)
calculate_platform_fee(payment_amount, author_monthly_revenue)
# → 0% если revenue ≤ 10K руб, 5% если > 10K руб

# Обработка платёжного webhook (Pseudocode.md)
handle_yukassa_webhook(payload, hmac_signature)
# → idempotency check → update subscription → grant access → notify

# Ежедневный renewal (Pseudocode.md)
process_daily_renewals()
# → upcoming notifications → ребиллинг → grace period → expiry

# Аналитика канала (Pseudocode.md)
aggregate_channel_analytics(channel_id, period_days)
# → MRR, subscribers, churn, ARPU, LTV — кэш Redis 1ч
```

## Security Rules

- **Платёжные данные**: НИКОГДА не хранить — только через ЮKassa hosted page
- **Webhook verification**: HMAC-SHA256 проверяется ПЕРВЫМ, до бизнес-логики
- **Idempotency**: все платёжные операции имеют idempotency_key
- **Secrets**: Bot token + ЮKassa keys только в Docker secrets, не в env файлах репо
- **Auth**: JWT (HS256) из Telegram initData verification; не хранить пароли
- **SQL**: только SQLAlchemy ORM (parameterized), никакого raw SQL с user input
- **Rate limiting**: 30 req/min per user_id (Nginx + Redis sliding window)
- См. `.claude/rules/secrets-management.md` для деталей по API ключам

## Parallel Execution Strategy

```
# Запускай независимые операции параллельно через Task tool
Task: "Run tests for bot/" | Task: "Run tests for api/" | Task: "Type check worker/"

# Для сложных фич — spawni специализированных агентов
@architect — системный дизайн новой фичи
@planner — детальный план реализации
@code-reviewer — проверка после реализации

# Никогда не блокируй на последовательных операциях если зависимости нет
```

## Swarm Agents

| Агент | Запуск | Назначение |
|-------|-------|-----------|
| @planner | `/plan [feature]` | Детальный план реализации из Pseudocode.md |
| @code-reviewer | После реализации | Edge cases, security, quality |
| @architect | Системный дизайн | Architecture.md + Solution_Strategy.md |

## Git Workflow

```
feat(bot): add subscription FSM flow          # новая фича
fix(worker): fix renewal idempotency bug      # баг фикс
refactor(api): extract payment service        # рефактор
test(bot): add registration FSM tests         # тесты
docs(api): update webhook endpoint docs       # документация
chore(infra): update docker-compose versions  # инфраструктура
```

## Available Agents

- `.claude/agents/planner.md` — планирование реализации фич
- `.claude/agents/code-reviewer.md` — code review с edge cases
- `.claude/agents/architect.md` — системный дизайн

## Available Skills

- `.claude/skills/project-context/` — доменный контекст TelePub
- `.claude/skills/coding-standards/` — Python/aiogram/FastAPI стандарты
- `.claude/skills/testing-patterns/` — паттерны тестирования (pytest + Gherkin)
- `.claude/skills/security-patterns/` — ЮKassa/Telegram security паттерны
- `.claude/skills/feature-navigator/` — навигация по roadmap
- `.claude/skills/sparc-prd-mini/` — планирование фич (SPARC)
- `.claude/skills/requirements-validator/` — валидация требований
- `.claude/skills/brutal-honesty-review/` — жёсткое code review

## Quick Commands

| Команда | Описание |
|---------|---------|
| `/start` | Bootstrap проекта из docs/ |
| `/feature [name]` | Полный lifecycle: plan → validate → implement → review |
| `/plan [feature]` | Быстрый план с сохранением в docs/plans/ |
| `/test [scope]` | Запуск/генерация тестов |
| `/deploy [env]` | Деплой на VPS (dev/staging/prod) |
| `/next` | Следующая фича из roadmap |
| `/go [feature]` | Smart pipeline: auto-select /plan, /feature |
| `/run [mvp\|all]` | Autonomous build loop |
| `/docs` | Генерация документации RU/EN |
| `/myinsights` | Захват и поиск инсайтов |
| `/review` | brutal-honesty-review текущих изменений |

## Development Insights

База знаний инсайтов: `myinsights/` (создаётся через `/myinsights`)

Критические инсайты перед разработкой:
- ЮKassa webhook HMAC проверять ДО парсинга payload (безопасность)
- aiogram FSM state хранить в Redis (не MemoryStorage) для горизонтального scale
- Telegram ChatMember ban + unban сразу = remove без блокировки (для re-subscribe)
- idempotency_key = новый UUID для каждого платёжного события (не subscription_id)
- Celery critical queue timeout: 30s; default: 5min; low: 30min

## Feature Development Lifecycle

```
1. /feature [name]    # или /go [name] для auto-selection
2. Plan:   sparc-prd-mini → 9 SPARC docs в docs/features/[name]/
3. Validate: requirements-validator → score ≥ 70, no BLOCKED
4. Implement: читай SPARC docs, параллельные Tasks, не галлюцинируй
5. Review: brutal-honesty-review → fix criticals
6. /next [id]          # закрыть в roadmap, unblock зависимые
```

## Feature Roadmap

Статус фич: `.claude/feature-roadmap.json`

**MVP (in_progress):**
- `F1` — Author Registration FSM (US-01)
- `F2` — Subscription Purchase Flow (US-02)
- `F3` — Access Management (US-03)

**Следующие:**
- `F4` — Author Payouts (US-04)
- `F5` — Basic Analytics Dashboard (US-05)

**v1.0 (planned):**
- `F9` — AI Analytics (MCP server)
- `F10` — Telegram Web App
- `F14` — Ad Exchange

## Implementation Plans

Активные планы: `docs/plans/`
Архивные планы: `docs/plans/archive/`

## Automation Commands

```
/go [feature]     → complexity score → /plan | /feature
/run mvp          → /start → /next → /go → repeat (MVP only)
/run all          → implement ALL features until done
/docs             → bilingual RU/EN documentation
```

Иерархия: `/run` → `/start` → `/next` → `/go` → `/plan` | `/feature`

## Resources

- PRD: `docs/PRD.md`
- Architecture: `docs/Architecture.md`
- Pseudocode: `docs/Pseudocode.md`
- Specification: `docs/Specification.md`
- BDD Tests: `docs/test-scenarios.md`
- Validation: `docs/validation-report.md`
- ЮKassa docs: https://yookassa.ru/developers
- aiogram docs: https://docs.aiogram.dev/
- FastAPI docs: https://fastapi.tiangolo.com/
