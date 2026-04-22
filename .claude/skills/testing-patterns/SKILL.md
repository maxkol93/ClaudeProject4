---
name: testing-patterns
description: TelePub testing patterns — pytest fixtures, testcontainers setup, aiogram handler mocking, payment webhook testing. Load when writing tests for any TelePub component.
version: "1.0"
maturity: production
---

# TelePub — Testing Patterns

Full strategy in `.claude/rules/testing.md`. This skill provides copy-paste test implementations.

## conftest.py — Base Fixtures

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from httpx import AsyncClient
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:16") as pg:
        yield pg

@pytest.fixture(scope="session")
def redis_container():
    with RedisContainer("redis:7") as redis:
        yield redis

@pytest_asyncio.fixture
async def session(postgres_container):
    url = postgres_container.get_connection_url().replace("psycopg2", "asyncpg")
    engine = create_async_engine(url)
    # Run migrations
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    
    async with AsyncSession(engine) as s:
        yield s
        await s.rollback()
    
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)

@pytest_asyncio.fixture
async def client(session):
    async with AsyncClient(app=app, base_url="http://test") as c:
        yield c
```

## Fixture: Valid ЮKassa Webhook Payloads

```python
# tests/fixtures/yukassa.py
import hmac
import hashlib
import json
from unittest.mock import patch

def make_yukassa_webhook(
    event: str,
    payment_id: str,
    status: str,
    amount: str = "299.00",
    payment_method_id: str = "pm_abc123",
    subscription_id: str = "sub_xyz",
) -> tuple[bytes, str]:
    """Returns (payload_bytes, valid_hmac_signature)."""
    payload = {
        "event": event,
        "object": {
            "id": payment_id,
            "status": status,
            "amount": {"value": amount, "currency": "RUB"},
            "payment_method": {"id": payment_method_id, "saved": True},
            "metadata": {"subscription_id": subscription_id},
        },
    }
    payload_bytes = json.dumps(payload).encode()
    secret = "test_yukassa_secret"
    signature = hmac.new(secret.encode(), payload_bytes, hashlib.sha256).hexdigest()
    return payload_bytes, signature

SUCCEEDED_WEBHOOK = make_yukassa_webhook(
    "payment.succeeded", "pay_001", "succeeded"
)
CANCELED_WEBHOOK = make_yukassa_webhook(
    "payment.canceled", "pay_002", "canceled"
)
```

## Test: Webhook Security

```python
# tests/integration/test_webhooks.py
import pytest
from httpx import AsyncClient

async def test_webhook_rejects_invalid_hmac(client: AsyncClient):
    payload_bytes, _ = make_yukassa_webhook("payment.succeeded", "pay_001", "succeeded")
    response = await client.post(
        "/webhooks/yukassa",
        content=payload_bytes,
        headers={
            "Content-Type": "application/json",
            "X-Webhook-Signature": "invalid_signature",
        },
    )
    assert response.status_code == 400
    assert "Invalid signature" in response.json()["detail"]

async def test_webhook_idempotency(client: AsyncClient, session):
    payload_bytes, signature = SUCCEEDED_WEBHOOK
    headers = {"Content-Type": "application/json", "X-Webhook-Signature": signature}
    
    # First call — creates payment
    r1 = await client.post("/webhooks/yukassa", content=payload_bytes, headers=headers)
    assert r1.status_code == 200
    
    # Second call — idempotency, no duplicate
    r2 = await client.post("/webhooks/yukassa", content=payload_bytes, headers=headers)
    assert r2.status_code == 200
    
    # Verify only one payment record exists
    payments = await get_payments_by_yukassa_id(session, "pay_001")
    assert len(payments) == 1
```

## Test: aiogram Handlers

```python
# tests/unit/test_registration.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from aiogram.types import Message, User, Chat
from aiogram.fsm.context import FSMContext
from bot.handlers.registration import cmd_register, handle_channel_input
from bot.states import RegistrationStates

def make_message(text: str, user_id: int = 123) -> Message:
    msg = MagicMock(spec=Message)
    msg.text = text
    msg.from_user = MagicMock(spec=User)
    msg.from_user.id = user_id
    msg.answer = AsyncMock()
    return msg

def make_state(data: dict = None) -> FSMContext:
    state = MagicMock(spec=FSMContext)
    state.set_state = AsyncMock()
    state.update_data = AsyncMock()
    state.get_data = AsyncMock(return_value=data or {})
    state.get_state = AsyncMock(return_value=None)
    return state

async def test_register_sets_awaiting_channel_state():
    message = make_message("/register")
    state = make_state()
    
    await cmd_register(message, state)
    
    state.set_state.assert_called_once_with(RegistrationStates.awaiting_channel)
    message.answer.assert_called_once()

async def test_channel_input_validates_at_prefix():
    message = make_message("finance_pro")  # missing @
    state = make_state()
    
    await handle_channel_input(message, state)
    
    state.set_state.assert_not_called()  # should not advance state
    assert "❌" in message.answer.call_args[0][0]
```

## Test: Business Logic (pure functions)

```python
# tests/unit/test_payments.py
import pytest
from decimal import Decimal
from shared.services.payments import calculate_platform_fee

@pytest.mark.parametrize("amount,revenue,expected_fee", [
    (Decimal("299"), Decimal("5000"), Decimal("0")),      # below threshold
    (Decimal("299"), Decimal("10001"), Decimal("14.95")), # above threshold: 299 * 0.05
    (Decimal("1000"), Decimal("15000"), Decimal("50.00")),
    (Decimal("299"), Decimal("10000"), Decimal("0")),     # at threshold (exclusive)
])
def test_platform_fee_calculation(amount, revenue, expected_fee):
    fee = calculate_platform_fee(amount, revenue)
    assert fee == expected_fee
```

## Test: Subscription Lifecycle

```python
# tests/integration/test_lifecycle.py
async def test_renewal_failure_moves_to_grace(session, mock_yukassa):
    """When renewal charge fails, subscription moves to grace period."""
    mock_yukassa.create.side_effect = Exception("Card declined")
    
    subscription = await create_test_subscription(session, status="active")
    
    await process_renewal(session, subscription.id)
    
    await session.refresh(subscription)
    assert subscription.status == "grace"
    assert subscription.grace_expires_at is not None

async def test_grace_period_expiry_removes_access(session, mock_telegram):
    """After grace period, subscriber is removed from channel."""
    sub = await create_test_subscription(session, status="grace", grace_expired=True)
    
    await process_expired_subscriptions(session)
    
    await session.refresh(sub)
    assert sub.status == "expired"
    mock_telegram.ban_chat_member.assert_called_once()
    mock_telegram.unban_chat_member.assert_called_once()
```

## Mocking Telegram API

```python
# tests/fixtures/telegram.py
from unittest.mock import AsyncMock, patch

@pytest.fixture
def mock_bot():
    with patch("bot.handlers.access.bot") as mock:
        mock.create_chat_invite_link = AsyncMock(return_value=MagicMock(invite_link="t.me/+abc123"))
        mock.ban_chat_member = AsyncMock()
        mock.unban_chat_member = AsyncMock()
        mock.send_message = AsyncMock()
        yield mock
```

## Performance Test Template

```python
# tests/performance/test_renewal_job.py
import time
import pytest

async def test_renewal_job_performance(session):
    """Daily renewal job must complete 1000 subscriptions in <30 seconds."""
    # Create 1000 expiring subscriptions
    await bulk_create_test_subscriptions(session, count=1000, status="active")
    
    start = time.monotonic()
    await process_daily_renewals(session)
    elapsed = time.monotonic() - start
    
    assert elapsed < 30.0, f"Renewal job took {elapsed:.1f}s (max 30s)"
```
