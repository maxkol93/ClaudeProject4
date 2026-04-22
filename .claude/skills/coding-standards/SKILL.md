---
name: coding-standards
description: TelePub Python coding patterns — aiogram 3.x, FastAPI, SQLAlchemy 2.0 async, Celery. Contains concrete code templates for recurring patterns. Load when implementing any Python module to write idiomatic code.
version: "1.0"
maturity: production
---

# TelePub — Coding Standards

Full rules in `.claude/rules/coding-style.md`. This skill provides copy-paste templates.

## SQLAlchemy Model Template

```python
# shared/models/[name].py
from __future__ import annotations
import uuid
from datetime import datetime
from decimal import Decimal
from sqlalchemy import String, DECIMAL, TIMESTAMPTZ, Boolean, ForeignKey, BigInteger
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.dialects.postgresql import UUID
from shared.database import Base

class Subscription(Base):
    __tablename__ = "subscriptions"
    __table_args__ = {"schema": "core"}

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    subscriber_telegram_id: Mapped[int] = mapped_column(BigInteger, nullable=False)
    plan_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("core.subscription_plans.id"))
    status: Mapped[str] = mapped_column(String(20), default="pending")
    yukassa_payment_method_id: Mapped[str | None] = mapped_column(String(255), nullable=True)
    started_at: Mapped[datetime | None] = mapped_column(TIMESTAMPTZ, nullable=True)
    expires_at: Mapped[datetime | None] = mapped_column(TIMESTAMPTZ, nullable=True)
    created_at: Mapped[datetime] = mapped_column(TIMESTAMPTZ, default=datetime.utcnow)

    plan: Mapped[SubscriptionPlan] = relationship("SubscriptionPlan", lazy="raise")
```

## Service Layer Template

```python
# shared/services/[name].py
from decimal import Decimal
from uuid import UUID
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload
from shared.models.subscription import Subscription
from shared.models.plan import SubscriptionPlan

PLATFORM_FEE_THRESHOLD = Decimal("10000")
PLATFORM_FEE_RATE = Decimal("0.05")

async def get_active_subscription(
    session: AsyncSession,
    subscriber_telegram_id: int,
    channel_id: UUID,
) -> Subscription | None:
    query = (
        select(Subscription)
        .join(SubscriptionPlan)
        .where(
            SubscriptionPlan.channel_id == channel_id,
            Subscription.subscriber_telegram_id == subscriber_telegram_id,
            Subscription.status == "active",
        )
        .options(selectinload(Subscription.plan))
    )
    return await session.scalar(query)

def calculate_platform_fee(
    payment_amount: Decimal,
    author_monthly_revenue: Decimal,
) -> Decimal:
    if author_monthly_revenue > PLATFORM_FEE_THRESHOLD:
        return (payment_amount * PLATFORM_FEE_RATE).quantize(Decimal("0.01"))
    return Decimal("0")
```

## FastAPI Router Template

```python
# api/routers/[name].py
from uuid import UUID
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from api.dependencies import get_current_author, get_session
from shared.models.author import Author

router = APIRouter(prefix="/channels", tags=["channels"])

@router.get("/{channel_id}/analytics", response_model=AnalyticsResponse)
async def get_analytics(
    channel_id: UUID,
    period: Literal["7d", "30d", "90d"] = "30d",
    current_author: Author = Depends(get_current_author),
    session: AsyncSession = Depends(get_session),
) -> AnalyticsResponse:
    channel = await get_channel(session, channel_id)
    if channel.author_id != current_author.id:
        raise HTTPException(403, "not_channel_owner")
    return await compute_analytics(session, channel_id, period)
```

## Webhook Handler Template

```python
# api/routers/webhooks.py
import hmac, hashlib
from fastapi import APIRouter, Request, HTTPException
from shared.config import settings

router = APIRouter(prefix="/webhooks")

@router.post("/yukassa")
async def handle_yukassa_webhook(request: Request) -> dict:
    payload_bytes = await request.body()
    signature = request.headers.get("X-Webhook-Signature", "")
    
    # ALWAYS FIRST — before any business logic
    if not verify_yukassa_hmac(payload_bytes, signature, settings.yukassa_secret):
        raise HTTPException(400, "Invalid signature")
    
    payload = await request.json()
    event = payload["event"]
    payment_obj = payload["object"]
    
    if event == "payment.succeeded":
        await process_payment_succeeded.apply_async(
            args=[payment_obj],
            queue="critical",
        )
    
    return {"status": "ok"}
```

## Celery Task Template

```python
# worker/tasks/[name].py
from celery import Task
from worker.celery_app import app
from shared.database import get_sync_session

@app.task(bind=True, max_retries=5, default_retry_delay=60)
def grant_channel_access(
    self: Task,
    subscriber_telegram_id: int,
    channel_id: str,
    subscription_id: str,
) -> None:
    try:
        # ... implementation
        pass
    except TelegramAPIError as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)

# Dispatch to critical queue
grant_channel_access.apply_async(
    args=[subscriber_id, channel_id, subscription_id],
    queue="critical",
)
```

## aiogram Handler Template

```python
# bot/handlers/[name].py
from aiogram import Router
from aiogram.filters import Command
from aiogram.types import Message, CallbackQuery
from aiogram.fsm.context import FSMContext
from bot.states import RegistrationStates

router = Router()

@router.message(Command("register"))
async def cmd_register(message: Message, state: FSMContext) -> None:
    await state.set_state(RegistrationStates.awaiting_channel)
    await message.answer(
        "Введите username вашего канала (например @my_channel):",
        reply_markup=cancel_keyboard(),
    )

@router.message(RegistrationStates.awaiting_channel)
async def handle_channel_input(message: Message, state: FSMContext) -> None:
    username = message.text.strip()
    if not username.startswith("@"):
        await message.answer("❌ Username должен начинаться с @")
        return
    
    await state.update_data(channel_username=username)
    await state.set_state(RegistrationStates.awaiting_bot_admin)
    await message.answer(f"Добавьте @TelePubBot в {username} как администратора...")
```

## FSM States Template

```python
# bot/states.py
from aiogram.fsm.state import State, StatesGroup

class RegistrationStates(StatesGroup):
    awaiting_channel = State()
    awaiting_bot_admin = State()
    awaiting_plan_setup = State()
    awaiting_yukassa = State()
```

## Database Session Template

```python
# shared/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from shared.config import settings

engine = create_async_engine(settings.database_url, echo=False, pool_size=10)
async_session_factory = async_sessionmaker(engine, expire_on_commit=False)

async def get_session() -> AsyncSession:
    async with async_session_factory() as session:
        yield session
```

## Config Template

```python
# shared/config.py
from pathlib import Path
from pydantic_settings import BaseSettings

def read_secret(name: str) -> str:
    p = Path(f"/run/secrets/{name}")
    if p.exists():
        return p.read_text().strip()
    import os
    return os.environ[name.upper()]

class Settings(BaseSettings):
    database_url: str = f"postgresql+asyncpg://telepub:{read_secret('postgres_password')}@postgres/telepub"
    redis_url: str = f"redis://:{read_secret('redis_password')}@redis:6379/0"
    bot_token: str = read_secret("telegram_bot_token")
    yukassa_shop_id: str = read_secret("yukassa_shop_id")
    yukassa_secret: str = read_secret("yukassa_secret_key")
    jwt_secret: str = read_secret("jwt_secret")
    jwt_algorithm: str = "HS256"
    jwt_expiry_hours: int = 24

settings = Settings()
```
