# Security Rules: TelePub

## Payment Security (CRITICAL)

- **НИКОГДА** не хранить данные банковских карт — только через ЮKassa hosted page
- **НИКОГДА** не логировать полные данные платёжных webhook'ов (только payment_id, status)
- HMAC-SHA256 верификация webhook'а — ПЕРВОЕ действие в обработчике, до любой бизнес-логики
- idempotency_key обязателен для каждого платёжного запроса (новый UUID, не subscription_id)
- При сомнении в статусе платежа — запрашивать ЮKassa API напрямую (reconciliation)

```python
# ПРАВИЛЬНО
def handle_webhook(payload, signature):
    if not verify_hmac(payload, signature, YUKASSA_SECRET):
        raise HTTPException(400, "Invalid signature")  # Первым делом!
    # ... затем бизнес-логика

# НЕПРАВИЛЬНО — никогда так не делать
def handle_webhook(payload, signature):
    subscription = db.find(payload['metadata']['subscription_id'])  # Нельзя до верификации!
    verify_hmac(...)
```

## Authentication & Authorization

- Telegram Bot: пользователь идентифицируется через `telegram_user_id` (верифицирован Telegram)
- Web Dashboard / TWA: JWT (HS256, 24ч), генерируется из Telegram initData HMAC verification
- Каждый API endpoint проверяет что `telegram_user_id` из JWT владеет запрашиваемым resource
- Admin операции (force-refund, suspend) — отдельный admin_token + IP whitelist

```python
# Проверка владения каналом в API
def require_channel_owner(channel_id: UUID, current_user: Author):
    channel = db.channels.get(channel_id)
    if channel.author_id != current_user.id:
        raise HTTPException(403, "not_channel_owner")
```

## Secrets Management

- Bot token: только в Docker secrets (`/run/secrets/telegram_bot_token`)
- ЮKassa secret: только в Docker secrets (`/run/secrets/yukassa_secret_key`)
- JWT secret: только в Docker secrets, минимум 32 байта случайных
- **НИКОГДА** не коммитить `.env` с реальными значениями
- `.env.example` — только placeholder значения, ок для git

## Input Validation

- Все входящие данные от пользователя — через Pydantic schemas с strict validation
- Telegram update объекты — aiogram парсит и типизирует автоматически
- Никакого raw SQL с user input — только SQLAlchemy ORM (parameterized)
- Размер payload webhook'а: проверять Content-Length до парсинга (max 1MB)

## Rate Limiting

- Bot: 30 req/min per telegram_user_id (Redis sliding window)
- API: 100 req/min per IP (Nginx)
- ЮKassa webhook endpoint: не rate-limit (IP whitelist ЮKassa)
- При превышении rate limit: HTTP 429, логировать user_id

## Data Protection

- PII хранение: только telegram_user_id (не имя, не username если не нужен)
- Данные карты: НЕ хранятся — ЮKassa токенизирует в yukassa_payment_method_id
- Backup: зашифрован AES-256 перед записью в S3/MinIO
- Логи: никогда не логировать telegram_user_id в plain text в production если не нужно

## Telegram Web App Security

- initData от Telegram проверяется HMAC-SHA256 с bot token перед выдачей JWT
- initData имеет expiry (auth_date) — проверять что не старше 1 часа

```python
def verify_telegram_init_data(init_data: str, bot_token: str) -> dict:
    parsed = parse_qs(init_data)
    received_hash = parsed.pop('hash')[0]
    data_check = '\n'.join(f'{k}={v[0]}' for k, v in sorted(parsed.items()))
    secret_key = hmac.new(b'WebAppData', bot_token.encode(), sha256).digest()
    expected_hash = hmac.new(secret_key, data_check.encode(), sha256).hexdigest()
    if not hmac.compare_digest(expected_hash, received_hash):
        raise ValueError("Invalid initData")
    return {k: v[0] for k, v in parsed.items()}
```

## Dependency Security

- `pip audit` в CI перед деплоем (проверка CVE)
- Pinned versions в requirements.txt (no `>=` без upper bound для critical deps)
- Обновлять aiogram, FastAPI, SQLAlchemy при security patch releases
