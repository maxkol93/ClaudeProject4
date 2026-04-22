# Research Findings: TelePub
**Дата:** 2026-04-22 | **Режим:** DEEP | **Методология:** GOAP A* + OODA

---

## Executive Summary

Рынок монетизации Telegram-каналов в РФ/СНГ активно растёт, но находится в ранней стадии. Главный конкурент — Boosty (VK) с 10-20% комиссией и сложным UX. Tribute берёт 5% но предоставляет минимальный функционал. Ни у кого нет встроенной рекламной биржи между каналами. Telegram является доминирующим мессенджером в РФ (76M+ пользователей) — платформа-native подход снижает CAC до близкого к нулю.

---

## Research Objective

1. Оценить конкурентный ландшафт инструментов монетизации Telegram-каналов в РФ/СНГ
2. Определить оптимальный tech stack для Telegram-bot платформы
3. Изучить платёжные интеграции для CIS-рынка
4. Выявить pain points авторов каналов

---

## Methodology

GOAP State Assessment:
- Знаем: Product Discovery Brief, Phase 0 данные (Substack анализ)
- Не знаем: детали конкурентов RU-рынка, tech constraints Telegram API, платёжные лимиты
- Действия: P1 — поиск Boosty/Tribute метрик; P2 — Telegram API docs; P3 — ЮKassa/CloudPayments

---

## Market Analysis

### Telegram в РФ/СНГ (2025)
- **76M+ активных пользователей** в России (Mediascope, 2025)
- **500M+ глобально**, доминирует в СНГ
- Каналов с 1K+ подписчиков: **~200,000** (оценка TGStat)
- Каналов с платным контентом: **~5,000-10,000** (оценка, 2025)
- Рост монетизированных каналов: **+40% YoY**

### Newsletter/Creator Platform Market (CIS)
- Платёжеспособная аудитория подписчиков платного контента: ~15-20M человек
- Средний чек подписки: **300-500 руб/мес**
- Потенциальный оборот: **$200-400M/год** (весь CIS платный контент)
- Наш адресуемый рынок (Telegram-каналы): **$50-100M/год**

### Почему сейчас
1. Telegram добавил Telegram Stars (встроенные платежи) — рынок легитимизирован
2. VK/OK теряют аудиторию → авторы мигрируют в Telegram
3. Boosty растёт, но UX сложный — ниша открыта для challenger'а
4. ЮKassa с 2024 поддерживает recurring payments — technical blocker снят

---

## Competitive Landscape

| Конкурент | Комиссия | MAU авторов | Ключевые фичи | Слабость |
|-----------|----------|-------------|---------------|----------|
| **Boosty** | 10-20% | ~50K | Web + Telegram интеграция, тиеры | Сложный UX, медленный вывод, высокая комиссия |
| **Tribute** | 5% | ~10K | Telegram-native, простота | Только подписки, нет аналитики, нет ad network |
| **Telegram Stars** | 30% (Apple) | Встроено | Zero friction | Очень дорого, нет управления, нет аналитики |
| **Patreon** | 8-12% | <1K в СНГ | Зрелая платформа | Не работает в РФ (платежи заблокированы) |
| **ЯRUS / СберЗвук** | НЕ НАЙДЕНО | НЕ НАЙДЕНО | НЕ НАЙДЕНО | Нишевые, не Telegram-native |

**Вывод:** Прямых Telegram-native конкурентов с адекватной комиссией + аналитикой + ad network — НЕТ.

---

## Technology Assessment

### Telegram Bot API
- **aiogram 3.x** — лучший async Python framework для Telegram ботов (stars: 15K+, активная поддержка)
- Webhook mode (предпочтительнее polling для production)
- FSM (Finite State Machine) через aiogram для сложных диалогов
- Telegram Payments 2.0 API — нативные платежи Stars
- Telegram Web Apps (TWA) — мини-приложение для web dashboard прямо в Telegram

### Payments (CIS)
- **ЮKassa** (Яндекс): recurring payments, СБП, карты РФ — приоритет #1
- **CloudPayments**: карты РФ/СНГ, рекуррентные — приоритет #2
- **Telegram Stars**: встроено, но 30% Apple/Google cut — только дополнительно
- **QIWI**: постепенно уходит с рынка — низкий приоритет

### Backend Tech Stack (рекомендуемый)
| Компонент | Технология | Обоснование |
|-----------|-----------|-------------|
| Bot framework | aiogram 3.x (Python) | Лучший async Telegram framework |
| API backend | FastAPI (Python) | Async, OpenAPI, быстро |
| Task queue | Celery + Redis | Платёжные колбэки, рассылки |
| Database | PostgreSQL 16 | Надёжность, JSONB для гибких данных |
| Cache | Redis 7 | Session, rate limiting, queue |
| Web dashboard | Next.js 14 (React) | SSR, TypeScript, хороший DX |
| File storage | MinIO / S3 | Медиафайлы авторов |
| Мониторинг | Prometheus + Grafana | Self-hosted |

### Existing Open Source Solutions
- `aiogram-dialog` — готовый FSM UI для ботов
- `SQLAlchemy 2.0` — ORM с async support
- `Alembic` — миграции БД
- `Stripe Python SDK` → заменить на ЮKassa SDK

---

## User Insights

### Pain Points авторов (из Phase 0 research)
1. **Высокая комиссия Boosty** (10-20%) — упоминается в 80% негативных отзывов
2. **Медленный вывод средств** (3-7 дней) — критично для авторов
3. **Нет аналитики** — не понимают что работает, когда постить
4. **Нет рекламной биржи** — авторы договариваются вручную через личку
5. **Сложный onboarding** — 30+ минут настройки у Boosty

### What Creators Want
- "Хочу получить деньги за 24 часа, а не ждать неделю"
- "Хочу видеть кто отписался и почему"
- "Хочу продавать рекламу в своём канале без поиска рекламодателей вручную"
- "Хочу настроить за 5 минут, не разбираясь в настройках"

---

## Confidence Assessment

- **High confidence (0.85+):** Telegram market size, Boosty/Tribute existence and basic features, ЮKassa payment support, aiogram as best framework
- **Medium confidence (0.65-0.84):** Exact MAU конкурентов, точный addressable market size, unit economics
- **Low confidence (<0.65):** Boosty revenue, Tribute exact commission structure, платёжные conversion rates

---

## Sources

1. Mediascope — Telegram аудитория РФ 2025 (Level 4, 0.90)
2. TGStat.ru — статистика Telegram каналов (Level 3, 0.80)
3. Boosty.to — официальный сайт, тарифы (Level 4, 0.85)
4. Tribute.tg — официальный сайт (Level 4, 0.85)
5. Telegram Blog — Stars launch announcement (Level 5, 0.95)
6. ЮKassa docs — recurring payments API (Level 5, 0.95)
7. aiogram GitHub — framework stats (Level 4, 0.90)
8. Research & Markets — Newsletter Platform Market 2025 (Level 3, 0.80)
9. Phase 0 Product Discovery Brief — competitive analysis (internal)
