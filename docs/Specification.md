# Specification: TelePub
**Дата:** 2026-04-22 | **Версия:** 1.0

---

## 1. System Actors

| Актор | Описание | Интерфейс |
|-------|----------|-----------|
| Author | Владелец Telegram-канала | Telegram Bot + Web Dashboard |
| Subscriber | Читатель, оформивший подписку | Telegram Bot |
| Advertiser | Покупатель рекламы (v2.0) | Web Dashboard |
| Platform Admin | Оператор TelePub | Admin Panel |
| ЮKassa | Платёжный провайдер | Webhook API |
| Telegram API | Внешний сервис | Bot API + TWA |

---

## 2. Functional Specifications

### FS-01: Author Registration & Onboarding

**Триггер:** /start или /register в боте

**Шаги:**
1. Бот запрашивает ссылку на канал (@channelname)
2. Бот проверяет что пользователь является admin канала (через getChat API)
3. Бот просит добавить @TelePubBot как admin с правами: Invite Users, Ban Users
4. Пользователь добавляет бота, бот верифицирует права
5. Бот предлагает создать закрытый (invite-link based) канал или использовать существующий
6. Настройка подписки: цена (руб/мес), описание, пробный период (0/3/7 дней)
7. Подключение ЮKassa: бот отправляет ссылку на ЮKassa OAuth
8. После авторизации ЮKassa — автор получает свой subscribe link

**Acceptance Criteria (Gherkin):**
```gherkin
Scenario: Успешная регистрация автора
  Given пользователь является admin Telegram-канала @mychannel
  When он отправляет /register и следует инструкциям бота
  Then через 5 минут его канал подключён к TelePub
  And он получает уникальную ссылку https://t.me/TelePubBot?start=channel_mychannel
  And на его dashboard отображается канал со статусом "Active"

Scenario: Бот не admin канала
  Given пользователь добавил бота без прав admin
  When бот проверяет права
  Then бот отправляет сообщение с инструкцией добавить права
  And процесс не продолжается до исправления
```

---

### FS-02: Subscription Purchase (Subscriber Flow)

**Триггер:** Подписчик переходит по ссылке https://t.me/TelePubBot?start=channel_{id}

**Шаги:**
1. Бот показывает описание канала, цену, пример контента (если есть)
2. Кнопки: "Подписаться" / "Попробовать бесплатно" (если настроен trial)
3. Выбор способа оплаты: Карта (ЮKassa) / Telegram Stars
4. ЮKassa: бот отправляет ссылку на платёжную форму
5. После успешного платежа: webhook от ЮKassa → платформа добавляет пользователя в канал через invite link
6. Подписчик получает приветственное сообщение и ссылку на вступление

**Acceptance Criteria:**
```gherkin
Scenario: Успешная оплата подписки
  Given читатель перешёл по subscribe link канала
  When он выбирает "Подписаться" и оплачивает через ЮKassa
  Then в течение 30 секунд он добавлен в закрытый канал
  And получает welcome-сообщение от автора
  And в dashboard автора появляется новый подписчик

Scenario: Оплата отклонена банком
  Given читатель пытается оплатить
  When банк отклоняет транзакцию
  Then бот сообщает об ошибке и предлагает другой способ оплаты
  And платёж не засчитывается
  And доступ не выдаётся
```

---

### FS-03: Subscription Lifecycle Management

**Процессы:**

| Событие | Триггер | Действие |
|---------|---------|----------|
| Renewal | За 3 дня до истечения | Уведомить подписчика, попытаться списать |
| Renewal success | Успешный ребиллинг | Продлить доступ на 30 дней |
| Renewal failed | Ошибка ребиллинга | Уведомить, дать 24ч grace period, удалить |
| Manual cancel | Подписчик отписался | Немедленно удалить из канала |
| Refund | Admin инициирует | Возврат через ЮKassa, удаление доступа |

**Acceptance Criteria:**
```gherkin
Scenario: Истечение подписки
  Given подписка истекает через 1 день
  When автоматический job запускается в 10:00 MSK
  Then система пытается списать средства через ЮKassa
  And при неудаче отправляет уведомление подписчику
  And через 24 часа grace period удаляет из канала
  And обновляет метрику churn в dashboard автора
```

---

### FS-04: Author Payouts

**Логика:**
- Баланс автора = Сумма успешных платежей × (1 - наша комиссия) × (1 - ЮKassa комиссия)
- Наша комиссия: 0% если writer_revenue ≤ 10,000 руб/мес; 5% если > 10,000 руб/мес
- ЮKassa комиссия: ~2.8% + 15 руб/транзакция (через агрегатор)
- Автовыплата: ежедневно в 10:00 MSK если баланс ≥ 1,000 руб
- Способы: банковская карта РФ, ЮMoney, расчётный счёт ИП/ООО

**Acceptance Criteria:**
```gherkin
Scenario: Автоматическая выплата
  Given баланс автора составляет 5,000 руб
  When срабатывает ежедневный payout job в 10:00 MSK
  Then инициируется перевод 5,000 руб на привязанную карту автора
  And транзакция отображается в истории выплат
  And автор получает Telegram-уведомление

Scenario: Баланс ниже минимума
  Given баланс автора составляет 800 руб
  When срабатывает payout job
  Then выплата не проводится
  And баланс накапливается до следующего дня
```

---

### FS-05: Analytics Dashboard

**Метрики MVP:**
- MRR (Monthly Recurring Revenue) — текущий и тренд
- Активных подписчиков — текущее значение
- Новых за период / Отписавшихся — с причиной (если известна)
- ARPU — средний чек
- LTV estimate — на основе retention curve
- Топ-3 поста по engagement (вовлечённости)

**Acceptance Criteria:**
```gherkin
Scenario: Просмотр аналитики
  Given автор открывает раздел Analytics в dashboard
  When страница загружается
  Then отображаются все 6 ключевых метрик
  And данные актуальны (lag не более 1 часа)
  And доступен экспорт в CSV за выбранный период
```

---

## 3. Non-Functional Specifications

### NFR-01: Производительность
- Bot API webhook: обработка обновления < 200ms (без внешних вызовов)
- Выдача/отзыв доступа к каналу: < 30 секунд
- Dashboard: Time to Interactive < 2 секунды
- Webhook от ЮKassa: обработка < 1 секунды с 200 OK ответом

### NFR-02: Безопасность
- Идемпотентность платёжных webhook'ов (повторная обработка = no-op)
- HMAC-верификация всех входящих webhook'ов
- Telegram Bot token хранится в Docker secrets / environment vault
- SQL injection prevention: только ORM запросы, parameterized
- Rate limiting: 30 req/min per user_id, 100 req/min per IP

### NFR-03: Данные
- Персональные данные (ФИО, email, номер карты) НЕ хранятся — всё через ЮKassa
- Telegram user_id хранится как primary identifier
- Хранение: PostgreSQL на серверах в РФ (HostKey/AdminVPS)
- Backup: ежедневный dump в зашифрованное S3-хранилище

### NFR-04: Доступность
- Telegram Bot: без downtime (webhook = stateless, несколько worker instances)
- База данных: primary + 1 replica read
- Деградация: если ЮKassa недоступна → показать уведомление, не падать

---

## 4. Domain Model

```
Author
  ├── telegram_user_id (PK)
  ├── channels: Channel[]
  ├── yukassa_account_id
  └── balance: Decimal

Channel
  ├── id (PK)
  ├── telegram_channel_id
  ├── author_id (FK → Author)
  ├── title, description
  ├── subscription_plans: SubscriptionPlan[]
  └── status: active | paused | suspended

SubscriptionPlan
  ├── id (PK)
  ├── channel_id (FK → Channel)
  ├── price_rub: Decimal
  ├── billing_interval: monthly | annual
  └── trial_days: Int

Subscription
  ├── id (PK)
  ├── subscriber_telegram_id
  ├── plan_id (FK → SubscriptionPlan)
  ├── status: active | grace | expired | cancelled
  ├── started_at, expires_at
  └── yukassa_subscription_id

Payment
  ├── id (PK, idempotency_key)
  ├── subscription_id (FK)
  ├── amount_rub: Decimal
  ├── platform_fee: Decimal
  ├── status: pending | success | failed | refunded
  └── yukassa_payment_id

Payout
  ├── id (PK)
  ├── author_id (FK)
  ├── amount_rub: Decimal
  ├── status: pending | completed | failed
  └── yukassa_payout_id
```

---

## 5. Integration Specifications

### ЮKassa Integration
- **Auth:** OAuth 2.0 + Shop ID + Secret Key
- **Payment creation:** POST /v3/payments с idempotency_key
- **Webhooks:** POST /webhooks/yukassa (HMAC-SHA256 verified)
- **Recurрent billing:** saved payment_method_id для ребиллинга
- **Payouts:** POST /v3/payouts

### Telegram Bot API
- **Mode:** Webhook (POST /telegram/webhook)
- **Framework:** aiogram 3.x
- **FSM:** MemoryStorage (Redis для production)
- **Inline keyboards:** для all interactive flows
- **ChatMember API:** add_chat_member, ban_chat_member для управления доступом

### Telegram Web App (TWA)
- URL: https://telepub.ru/app
- Данные: initData от Telegram (верифицированы HMAC)
- Auth: JWT на основе telegram_user_id
