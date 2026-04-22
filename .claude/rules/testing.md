# Testing Strategy: TelePub

## Test pyramid

```
          [E2E]           — Playwright (happy paths, critical flows)
        [Integration]     — pytest + testcontainers (real DB + Redis)
      [Unit]              — pytest (business logic, pure functions)
    [Type Check]          — mypy --strict (all modules)
  [Lint]                  — ruff (style + common bugs)
```

## Running tests

```bash
# Full suite
pytest tests/ -v --cov=. --cov-report=term-missing

# Unit only (fast, no external deps)
pytest tests/unit/ -v -x

# Integration (requires Docker)
pytest tests/integration/ -v --tb=short

# Single file
pytest tests/unit/test_payments.py -v -k "test_hmac"

# Type check
mypy bot/ api/ worker/ shared/ --strict --ignore-missing-imports

# Lint
ruff check . --fix && ruff format .
```

## Coverage targets

| Module | Min coverage |
|--------|-------------|
| `shared/services/` | 90% |
| `worker/tasks/` | 85% |
| `bot/handlers/` | 80% |
| `api/routers/` | 80% |
| Overall | 80% |

## Unit test patterns

```python
# tests/unit/test_payments.py
import pytest
from decimal import Decimal
from shared.services.payments import calculate_platform_fee

class TestPlatformFee:
    def test_below_threshold_zero_fee(self):
        fee = calculate_platform_fee(
            payment_amount=Decimal("299"),
            author_monthly_revenue=Decimal("5000"),
        )
        assert fee == Decimal("0")

    def test_above_threshold_five_percent(self):
        fee = calculate_platform_fee(
            payment_amount=Decimal("299"),
            author_monthly_revenue=Decimal("15000"),
        )
        assert fee == Decimal("14.95")  # 299 * 0.05

    def test_exactly_at_threshold(self):
        fee = calculate_platform_fee(
            payment_amount=Decimal("299"),
            author_monthly_revenue=Decimal("10000"),
        )
        assert fee == Decimal("0")  # threshold is exclusive
```

## Integration test patterns

```python
# tests/integration/test_webhook.py
import pytest
import pytest_asyncio
from httpx import AsyncClient
from testcontainers.postgres import PostgresContainer

@pytest_asyncio.fixture
async def client(postgres_container):
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

async def test_yukassa_webhook_invalid_hmac(client):
    response = await client.post(
        "/webhooks/yukassa",
        json={"event": "payment.succeeded", "object": {"id": "pay_123"}},
        headers={"X-Webhook-Signature": "invalid"},
    )
    assert response.status_code == 400
    assert "Invalid signature" in response.json()["detail"]
```

## Fixtures

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

@pytest_asyncio.fixture
async def session(postgres_url):
    engine = create_async_engine(postgres_url)
    async with AsyncSession(engine) as session:
        yield session
        await session.rollback()

@pytest.fixture
def valid_yukassa_payload():
    return {
        "event": "payment.succeeded",
        "object": {
            "id": "24b94598-000f-5000-9000-1b68e7b15f3f",
            "status": "succeeded",
            "amount": {"value": "299.00", "currency": "RUB"},
            "payment_method": {"id": "pm_abc123"},
            "metadata": {"subscription_id": "sub_xyz"},
        }
    }
```

## Critical test scenarios (from test-scenarios.md)

Always test these flows:

1. **HMAC rejection**: forged webhook → 400 response, no DB changes
2. **Idempotency**: duplicate webhook → 200, no duplicate subscription
3. **Access grant**: payment succeeded → subscriber in channel within 30s
4. **Renewal failure**: card declined → grace period → expiry
5. **Race condition**: simultaneous renew + cancel → exactly one state

## Mocking policy

- **NEVER** mock PostgreSQL in integration tests (use testcontainers)
- **NEVER** mock ЮKassa webhooks for idempotency tests (use fixture payloads)
- Mock Telegram API calls (aiogram BotAPI) in unit tests via `AsyncMock`
- Mock Redis in unit tests, use real Redis in integration tests

## Performance benchmarks (from Refinement.md NFRs)

```python
# tests/performance/test_latency.py
async def test_bot_response_latency(benchmark):
    # Bot must respond in <500ms p95
    result = benchmark(handle_start_command, mock_message)
    assert benchmark.stats["mean"] < 0.5

async def test_access_grant_latency(benchmark):
    # Access grant must complete in <30s
    result = benchmark(grant_channel_access, subscriber_id, channel_id)
    assert benchmark.stats["max"] < 30.0
```
