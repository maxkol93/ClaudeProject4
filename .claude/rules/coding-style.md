# Coding Style: TelePub

## Python (Bot + API + Worker)

### Общие правила
- Python 3.12+, type hints везде
- `ruff` для linting и formatting (замена flake8 + black + isort)
- `mypy --strict` для type checking (CI блокирует при ошибках)
- Максимальная длина строки: 100 символов

### Именование
```python
# Файлы: snake_case
bot/handlers/subscription.py
api/routers/channels.py
worker/tasks/payments.py

# Классы: PascalCase
class SubscriptionService:
class YuKassaWebhookHandler:

# Функции/переменные: snake_case
async def handle_subscription_created(event: SubscriptionCreated) -> None:
author_monthly_revenue = await get_monthly_revenue(author_id)

# Константы: UPPER_SNAKE_CASE
MINIMUM_PAYOUT_RUB = Decimal("1000")
PLATFORM_FEE_THRESHOLD_RUB = Decimal("10000")
PLATFORM_FEE_RATE = Decimal("0.05")
```

### Async правила
```python
# ВСЕ I/O операции — async
async def get_subscription(subscription_id: UUID) -> Subscription:
    async with AsyncSession(engine) as session:
        return await session.get(Subscription, subscription_id)

# НЕТ blocking calls в async контексте
# ПЛОХО:
result = requests.get(url)  # блокирует event loop!
# ХОРОШО:
async with httpx.AsyncClient() as client:
    result = await client.get(url)
```

### SQLAlchemy 2.0 Async
```python
# Всегда использовать async_session
from sqlalchemy.ext.asyncio import AsyncSession

async def create_subscription(session: AsyncSession, data: SubscriptionCreate) -> Subscription:
    sub = Subscription(**data.model_dump())
    session.add(sub)
    await session.commit()
    await session.refresh(sub)
    return sub

# Никогда lazy loading в async — использовать selectinload/joinedload
query = select(Subscription).options(selectinload(Subscription.plan))
```

### aiogram 3.x
```python
# Handlers через декораторы на Router
from aiogram import Router
router = Router()

@router.message(Command("start"))
async def cmd_start(message: Message, state: FSMContext) -> None:
    await state.set_state(RegistrationStates.awaiting_channel)
    await message.answer("...")

# FSM States — строго через StatesGroup
class RegistrationStates(StatesGroup):
    awaiting_channel = State()
    awaiting_bot_admin = State()
    awaiting_plan_setup = State()
    awaiting_yukassa = State()
```

### Import ordering (ruff isort)
```python
# 1. Standard library
import hashlib
from datetime import datetime
from decimal import Decimal

# 2. Third-party
from aiogram import Bot, Dispatcher
from fastapi import FastAPI, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

# 3. Local (absolute paths)
from telepub.core.models import Author, Subscription
from telepub.services.payment import YuKassaService
```

## FastAPI Patterns

```python
# Dependency injection для сессии и auth
async def get_current_author(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_session),
) -> Author:
    ...

# Response models всегда явные
@router.get("/channels/{channel_id}/analytics", response_model=AnalyticsResponse)
async def get_analytics(
    channel_id: UUID,
    period: Literal["7d", "30d", "90d"] = "30d",
    current_author: Author = Depends(get_current_author),
    session: AsyncSession = Depends(get_session),
) -> AnalyticsResponse:
    ...
```

## Celery Tasks

```python
# Всегда с retry и max_retries
@app.task(bind=True, max_retries=5, default_retry_delay=60)
def grant_channel_access(self, subscriber_telegram_id: int, channel_id: int) -> None:
    try:
        ...
    except TelegramAPIError as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)

# Queues: critical, default, low — указывай явно
grant_channel_access.apply_async(args=[...], queue="critical")
```

## Known Gotchas

### aiogram / Telegram
- `ban_chat_member` + немедленный `unban_chat_member` = удаление без блокировки. Это правильно для re-subscribe.
- FSM state в MemoryStorage теряется при перезапуске бота. В production — только RedisStorage.
- Telegram rate limit: 30 сообщений/секунду на бота. При массовых рассылках — throttling через aiogram middleware.
- `create_chat_invite_link` с `member_limit=1` — одноразовая ссылка. После использования создавать новую.

### ЮKassa
- idempotency_key должен быть НОВЫМ UUID для каждой попытки платежа, не переиспользовать.
- Webhook может прийти несколько раз — всегда проверять `payment_id` на дубликат.
- `payment_method_id` из succeeded webhook сохранять — нужен для ребиллинга.
- Статус `pending` у webhook — ждать `succeeded`/`canceled`, не считать успехом.

### SQLAlchemy Async
- Lazy loading не работает в async контексте — всегда `selectinload` или `joinedload`.
- `session.refresh(obj)` нужен после `commit()` чтобы получить server-generated поля (id, created_at).
- `SELECT FOR UPDATE` для pessimistic locking при race conditions (renewal + cancel).

### PostgreSQL / Alembic
- Всегда создавать индекс для `expires_at` в subscriptions — daily renewal job делает range scan.
- `DECIMAL(12,2)` для денег, никогда `FLOAT` (потери точности).
- Миграции: никогда не удалять колонки без data migration. Сначала deprecate, потом удалить.

### Docker / Deploy
- Health check для bot и api через `/api/health` endpoint — Docker Compose depends_on.
- Celery worker не имеет HTTP endpoint — health check через `celery inspect ping`.
- Secrets через Docker secrets (`/run/secrets/`), не через env vars в compose file.
