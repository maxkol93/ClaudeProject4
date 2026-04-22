# Refinement: TelePub
**Дата:** 2026-04-22

---

## 1. Edge Cases Matrix

| Сценарий | Входные данные | Ожидаемое поведение | Обработка |
|----------|---------------|---------------------|-----------|
| Автор добавляет бота без прав admin | Channel без прав Invite/Ban | Бот сообщает какие права нужны, ждёт | Graceful prompt |
| Подписчик уже в канале (переподписка) | Subscription expired, user still in channel | Не добавлять повторно, просто обновить статус | Check membership before add |
| ЮKassa webhook пришёл дважды | Дублированный payment_id | Idempotency key → no-op для второго | Idempotency check |
| Автор удалил закрытый канал | Channel deleted in Telegram | Все подписки отменить, авторов уведомить | Polling channel status |
| Банк отклонил, но деньги списаны | Partial payment state | Reconciliation с ЮKassa API каждые 5 мин | Job: verify pending payments |
| Бот забанен в канале | Bot kicked from channel | Все подписки на паузу, автору уведомление | ChatMember webhook event |
| Подписчик заблокировал бота | Bot can't message subscriber | Catch TelegramForbiddenError, mark user as blocked | Try/except → update flag |
| Одновременный renewal + ручная отписка | Race condition | Pessimistic lock через SELECT FOR UPDATE | DB transaction |
| Автор снижает цену | Существующие подписчики | Продолжают по старой цене (grandfathering) | Plan versioning |
| Большая выплата (>600K руб) | Лимит ЮKassa | Разбить на несколько транзакций | Split if > 600K |
| Пользователь из страны не-РФ | Иностранная карта | Показать предупреждение, попробовать | CloudPayments fallback |
| Telegram API недоступен | 502 from Telegram | Retry с backoff, задача в очередь | Queue + retry |

---

## 2. Testing Strategy

### Unit Tests (pytest)
**Coverage target:** 80% для бизнес-логики

**Критические пути:**
- `calculate_platform_fee()` — все граничные случаи (0, 9999, 10000, 10001 руб)
- `handle_yukassa_webhook()` — все статусы (succeeded, canceled, refunded)
- `process_daily_renewals()` — grace period логика
- JWT генерация и верификация
- HMAC верификация webhook'ов

```python
# Пример
def test_calculate_platform_fee_below_threshold():
    fee = calculate_platform_fee(payment_amount=5000, author_monthly_revenue=8000)
    assert fee == 0  # 0% если revenue <= 10K

def test_calculate_platform_fee_above_threshold():
    fee = calculate_platform_fee(payment_amount=5000, author_monthly_revenue=15000)
    assert fee == 250  # 5% от 5000
```

### Integration Tests
**Coverage:** Ключевые flows end-to-end

- Регистрация автора → webhook верификация → подключение ЮKassa
- Оплата подписчика → webhook → доступ к каналу (mock Telegram API)
- Ребиллинг → успех → продление; ребиллинг → неудача → grace period
- Выплата автора → mock ЮKassa payout

**Инструменты:** pytest-asyncio, respx (mock httpx), testcontainers (PostgreSQL, Redis)

### E2E Tests (Playwright)
**Coverage:** Happy path только для critical flows

- Автор: открывает dashboard → видит метрики → инициирует выплату
- TWA: открывается в Telegram → данные отображаются корректно

### Performance Tests (Locust)
**Targets:**
- Bot webhook: 1,000 req/sec при <200ms p95
- Payment webhook processing: 100 req/sec при <1s p95
- Dashboard API: 500 req/sec при <500ms p95

```python
# locustfile.py target
class BotWebhookUser(HttpUser):
    @task
    def send_update(self):
        self.client.post("/telegram/webhook", json=mock_telegram_update())
```

---

## 3. Test Cases (Gherkin)

```gherkin
Feature: Subscription Payment Flow

  Scenario: Happy path — new subscriber
    Given channel "finance_expert" exists and is connected to TelePub
    And subscriber user_id=12345 has not subscribed before
    When subscriber sends /subscribe to bot
    And chooses "Подписаться за 500 руб/мес"
    And completes ЮKassa payment
    Then webhook arrives with status "succeeded"
    And subscriber is added to private channel within 30 seconds
    And author's dashboard shows +1 subscriber
    And author's balance increases by 500 - fees

  Scenario: Payment webhook — duplicate
    Given payment yukassa_id="pay_abc123" was processed successfully
    When same webhook arrives again with yukassa_id="pay_abc123"
    Then system returns HTTP 200
    And no duplicate payment record created
    And subscriber's subscription unchanged

  Scenario: Renewal failure → grace period
    Given active subscription expires_at = NOW() - 1 second
    And ребиллинг через ЮKassa возвращает status="canceled"
    When daily renewal job runs
    Then subscription status changes to "grace"
    And subscriber receives notification "Не удалось списать оплату"
    And subscriber remains in channel for 24 hours
    After 24 hours without payment:
    Then subscription status changes to "expired"
    And subscriber is removed from channel

Feature: Author Payout

  Scenario: Successful auto-payout
    Given author balance = 5000 RUB
    And author has verified payout card
    When daily payout job runs at 10:00 MSK
    Then ЮKassa payout initiated for 5000 RUB
    And author balance reset to 0
    And author receives Telegram notification
    And payout record created with status="processing"

  Scenario: Balance below minimum
    Given author balance = 750 RUB
    When daily payout job runs
    Then no payout initiated
    And balance remains 750 RUB
    And no notification sent

Feature: Author Registration

  Scenario: Successful registration
    Given user is admin of Telegram channel @mychannel
    And bot is added as admin with Invite/Ban permissions
    When user sends /register and completes all steps
    Then channel is connected to TelePub
    And user receives subscribe link: t.me/TelePubBot?start=channel_mychannel
    And dashboard shows channel status = "active"

  Scenario: Bot lacks admin rights
    Given bot is added to channel without admin rights
    When user tries to verify channel
    Then bot sends error message with required permissions
    And registration pauses until rights are granted
```

---

## 4. Performance Optimizations

### Database
- **Индексы:** expires_at (for daily renewal queries), subscriber_telegram_id, yukassa_payment_id
- **Partitioning:** таблица payments — по месяцу (после 1M записей)
- **Connection pooling:** asyncpg connection pool (min=5, max=20 per service)
- **Read replica:** analytics queries → replica, writes → primary
- **Vacuum:** настроить autovacuum для subscriptions (высокий update rate)

### Caching (Redis)
- Channel metadata: TTL 5 минут (часто читается при каждом update)
- Author dashboard stats: TTL 1 час (дорогая агрегация)
- Rate limiting: sliding window counter per user_id
- FSM state: в Redis (не в памяти — для горизонтального масштабирования)

### Bot Performance
- Обработка webhook'ов: async everywhere (никаких blocking I/O)
- Telegram API calls: через aiogram's built-in throttling
- Тяжёлые операции (выдача доступа) → в Celery worker (не в webhook handler)

### Celery Queue Priority
```
critical:  платёжные webhook'и, выдача/отзыв доступа (timeout: 30s)
default:   уведомления авторам/подписчикам (timeout: 5min)
low:       аналитика, агрегация, email reports (timeout: 30min)
```

---

## 5. Security Hardening

### Input Validation
- Все Telegram update'ы: aiogram автоматически парсит и типизирует
- API endpoints: Pydantic schemas с strict validation
- SQL: только SQLAlchemy ORM (parameterized queries), никакого raw SQL с user input
- ЮKassa webhook HMAC: проверяется ПЕРВЫМ делом, до любой бизнес-логики

### Rate Limiting (Nginx + Redis)
```nginx
# Per user rate limits
limit_req_zone $telegram_user_id zone=bot_user:10m rate=30r/m;
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;

# Payment webhook — не rate limit (ЮKassa IP whitelist)
geo $yukassa_ip {
    default 0;
    185.71.76.0/27 1;  # ЮKassa IPs
}
```

### Audit Logging
- Все платёжные операции: structured log с user_id, amount, action
- Все выплаты: отдельная audit таблица с immutable records
- Admin actions: отдельный audit log

### Secrets Management
```yaml
# docker-compose.yml
secrets:
  telegram_bot_token:
    file: ./secrets/telegram_bot_token.txt
  yukassa_secret_key:
    file: ./secrets/yukassa_secret.txt
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
```

---

## 6. Technical Debt Items

| Пункт | Приоритет | Когда | Описание |
|-------|-----------|-------|----------|
| FSM in Redis | Medium | v1.0 | Сейчас MemoryStorage в dev → Redis в prod |
| Payment reconciliation | High | MVP | Cron job сверяет pending платежи с ЮKassa каждые 5 мин |
| CloudPayments fallback | Medium | v1.0 | Второй провайдер для отказоустойчивости |
| Analytics ML | Low | v2.0 | Сейчас простая агрегация → ML модели для churn prediction |
| Multi-language | Low | v2.0 | Сейчас только RU, в будущем KZ, UA локализации |
| Database partitioning | Low | v2.0 | Нужно при >1M payments |
