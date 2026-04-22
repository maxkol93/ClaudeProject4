# PRD: TelePub — Telegram-Native Monetization Platform
**Версия:** 1.0 | **Дата:** 2026-04-22 | **Статус:** Draft

---

## 1. Overview

### Product Vision
TelePub — Telegram-native платформа монетизации каналов для авторов РФ/СНГ. Позволяет авторам принимать платежи за закрытый контент, управлять подписчиками и продавать рекламу — всё через Telegram-бот без выхода из мессенджера.

### Problem Statement
Авторы Telegram-каналов в РФ/СНГ не имеют удобного, дешёвого и функционального инструмента монетизации. Boosty берёт 10-20%, Tribute — примитивен, Telegram Stars — 30% Apple cut.

### Solution
Telegram-bot + Web dashboard с 0% комиссии до 10K руб/мес, 5% после — с AI-аналитикой, быстрыми выплатами (24ч) и рекламной биржей.

---

## 2. Target Users

### Persona 1: Активный автор (Primary)
- **Кто:** Ведёт Telegram-канал 1K-100K подписчиков в нише (финансы, tech, lifestyle, образование)
- **Боль:** Зарабатывает на рекламе нерегулярно, хочет стабильный доход от подписки
- **Цель:** Монетизировать 3-5% аудитории как платных подписчиков
- **Техническая грамотность:** Средняя (умеет настраивать боты)

### Persona 2: Читатель/Подписчик (Secondary)
- **Кто:** Читает 5-10 Telegram-каналов, готов платить за качественный контент
- **Боль:** Нет удобного способа платить за контент в Telegram без лишних шагов
- **Цель:** Получить эксклюзивный контент от любимого автора

### Persona 3: Рекламодатель (Tertiary, v2.0)
- **Кто:** Малый/средний бизнес, другие авторы каналов
- **Боль:** Поиск подходящих каналов для рекламы — ручной и непрозрачный процесс
- **Цель:** Эффективно размещать рекламу в тематических каналах с измеримым ROI

---

## 3. Feature Matrix

### MVP (Month 1-3)
| # | Фича | Описание | Priority |
|---|------|----------|----------|
| F1 | Регистрация автора | Бот: добавить канал, верификация владельца, настройка подписки | P0 |
| F2 | Настройка подписки | Установить цену (руб/мес), описание, пробный период | P0 |
| F3 | Приём платежей | ЮKassa (карты РФ, СБП) + Telegram Stars | P0 |
| F4 | Управление доступом | Автоматическое добавление/удаление из закрытого канала | P0 |
| F5 | Подписка читателя | Bot: найти канал, оформить подписку, оплатить | P0 |
| F6 | Базовая аналитика | Dashboard: MRR, подписчики, churn, топ-посты | P1 |
| F7 | Выплаты | Автовыплата на карту / ЮMoney за 24ч при балансе >1K руб | P0 |
| F8 | Уведомления автора | Новый подписчик, отписка, успешная выплата | P1 |

### v1.0 (Month 4-6)
| # | Фича | Описание | Priority |
|---|------|----------|----------|
| F9 | AI-аналитика | Лучшее время публикации, тема-предсказатель, churn risk | P1 |
| F10 | Telegram Web App | Мини-приложение внутри Telegram: полный dashboard | P1 |
| F11 | Промокоды | Скидочные коды для роста аудитории | P2 |
| F12 | Реферальная программа | Авторы приглашают авторов → скидка на комиссию | P2 |
| F13 | Уровни подписки | Бесплатный / Базовый / Премиум тиеры | P2 |

### v2.0 (Month 7-12)
| # | Фича | Описание | Priority |
|---|------|----------|----------|
| F14 | Рекламная биржа | Авторы публикуют слоты, рекламодатели покупают | P1 |
| F15 | Мультиканальность | Управление несколькими каналами из одного аккаунта | P2 |
| F16 | API | REST API для агентств и интеграций | P3 |
| F17 | CloudPayments | Второй платёжный провайдер | P1 |

---

## 4. User Stories (MVP)

### US-01: Регистрация автора
```
As a Telegram channel author,
I want to connect my channel to TelePub in under 5 minutes,
So that I can start accepting paid subscriptions immediately.

Acceptance Criteria:
Given I am a Telegram channel admin
When I start /register command in TelePub bot
Then bot guides me through: add bot as admin → verify ownership → set price → connect ЮKassa
And the entire flow takes <5 minutes
And I receive confirmation message with my dashboard link
```

### US-02: Оформление подписки читателем
```
As a reader,
I want to subscribe to a paid Telegram channel easily,
So that I get immediate access to exclusive content.

Acceptance Criteria:
Given I find TelePub-powered channel
When I click "Subscribe" button or /subscribe command
Then I see price and what's included
And I can pay via card (ЮKassa) or Telegram Stars
And within 30 seconds after payment I am added to private channel
And I receive welcome message from author
```

### US-03: Управление доступом
```
As the platform,
I want to automatically remove expired subscribers from private channels,
So that authors don't have to manually manage access.

Acceptance Criteria:
Given a subscriber's payment fails or subscription expires
When the renewal date passes without successful payment
Then subscriber is automatically removed from private channel within 1 hour
And subscriber receives notification with renewal link
And author's dashboard updates in real-time
```

### US-04: Выплата автору
```
As a channel author,
I want to receive my earnings within 24 hours,
So that I have predictable cash flow.

Acceptance Criteria:
Given my balance exceeds 1,000 RUB
When automatic payout trigger fires (daily at 10:00 MSK)
Then funds transfer to my linked bank card/ЮMoney within 24 hours
And I receive Telegram notification with payout amount
And transaction is recorded in my dashboard
```

### US-05: Базовая аналитика
```
As a channel author,
I want to see my key metrics in one place,
So that I understand my channel's financial health.

Acceptance Criteria:
Given I am logged in to TelePub dashboard
When I open Analytics section
Then I see: MRR, total subscribers, new this month, churned this month, avg subscription duration
And data updates every hour
And I can export to CSV
```

---

## 5. Non-Functional Requirements

### Performance
- Bot response time: <500ms for 95th percentile
- Payment processing: <3 seconds end-to-end (excluding bank)
- Dashboard load time: <2 seconds
- Channel access grant/revoke: <30 seconds after payment

### Security
- All payment data handled exclusively by ЮKassa (PCI DSS compliant) — мы не храним данные карт
- Subscriber data encrypted at rest (AES-256)
- Bot tokens stored in environment variables / Vault
- Admin actions require 2FA confirmation via Telegram
- Rate limiting: 30 requests/min per user

### Scalability
- Supports 10,000 concurrent active subscriptions (MVP)
- Supports 100,000 (v1.0) — горизонтальное масштабирование bot workers
- Database: read replicas for analytics queries

### Reliability
- Uptime SLA: 99.5% (MVP), 99.9% (v1.0)
- Payment webhook processing: at-least-once delivery с idempotency keys
- Graceful degradation: если ЮKassa недоступна → queue + retry, уведомить автора

### Compliance
- Соответствие 152-ФЗ (персональные данные): хранение на серверах РФ или с согласием
- Соответствие требованиям ЦБ РФ как платёжный агент (через ЮKassa как лицензированный агрегатор)
- GDPR: не применимо (фокус РФ/СНГ), но best practices

---

## 6. Success Metrics

### Business KPIs
| Метрика | MVP (M3) | v1.0 (M6) | v2.0 (M12) |
|---------|----------|-----------|------------|
| Активных авторов | 100 | 500 | 2,000 |
| Платных подписок (total) | 2,000 | 20,000 | 100,000 |
| MRR платформы | 30K руб | 300K руб | 1.5M руб |
| Avg выплата автору/мес | 15K руб | 20K руб | 25K руб |
| Churn авторов | <10%/мес | <7%/мес | <5%/мес |

### Product KPIs
| Метрика | Target |
|---------|--------|
| Onboarding completion rate | >70% |
| Time to first payment received | <48 часов |
| Bot response time p95 | <500ms |
| Payment success rate | >95% |
| Support tickets per 100 авторов | <5/мес |

---

## 7. Out of Scope (MVP)

- Поддержка YouTube / VK / других платформ (только Telegram)
- Мобильное приложение (только bot + TWA)
- Международные платежи (только РФ/СНГ карты)
- Встроенный редактор контента (авторы постят через Telegram напрямую)
- Content moderation (ответственность автора, по ToS)
