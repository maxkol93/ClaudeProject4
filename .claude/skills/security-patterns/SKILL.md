---
name: security-patterns
description: TelePub security implementation patterns — HMAC webhook verification, JWT auth, Telegram initData validation, idempotency checks. Load when implementing any payment handler, API endpoint, or Telegram integration.
version: "1.0"
maturity: production
---

# TelePub — Security Patterns

Full rules in `.claude/rules/security.md`. This skill provides ready-to-use security implementations.

## Pattern 1: ЮKassa Webhook Verification

```python
import hmac
import hashlib
from fastapi import Request, HTTPException

async def require_valid_yukassa_signature(request: Request) -> bytes:
    """Must be called BEFORE any business logic in webhook handler."""
    payload_bytes = await request.body()
    signature = request.headers.get("X-Webhook-Signature", "")
    secret = read_secret("yukassa_secret_key")
    
    expected = hmac.new(
        secret.encode("utf-8"),
        payload_bytes,
        hashlib.sha256,
    ).hexdigest()
    
    if not hmac.compare_digest(expected, signature):
        raise HTTPException(status_code=400, detail="Invalid signature")
    
    return payload_bytes

# Usage in handler:
@router.post("/webhooks/yukassa")
async def webhook(
    request: Request,
    payload_bytes: bytes = Depends(require_valid_yukassa_signature),
    session: AsyncSession = Depends(get_session),
) -> dict:
    # Signature already verified by dependency
    payload = json.loads(payload_bytes)
    ...
```

## Pattern 2: JWT Authentication

```python
import jwt
from datetime import datetime, timedelta
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def create_jwt(telegram_user_id: int, channel_ids: list[str]) -> str:
    payload = {
        "sub": str(telegram_user_id),
        "telegram_user_id": telegram_user_id,
        "channels": channel_ids,
        "exp": datetime.utcnow() + timedelta(hours=24),
        "iat": datetime.utcnow(),
    }
    return jwt.encode(payload, read_secret("jwt_secret"), algorithm="HS256")

async def get_current_author(
    token: str = Depends(oauth2_scheme),
    session: AsyncSession = Depends(get_session),
) -> Author:
    try:
        payload = jwt.decode(
            token,
            read_secret("jwt_secret"),
            algorithms=["HS256"],
        )
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Invalid token")
    
    author = await get_author_by_telegram_id(session, int(payload["telegram_user_id"]))
    if not author:
        raise HTTPException(401, "Author not found")
    return author
```

## Pattern 3: Channel Ownership Check

```python
async def require_channel_owner(
    channel_id: UUID,
    current_author: Author = Depends(get_current_author),
    session: AsyncSession = Depends(get_session),
) -> Channel:
    """Add as dependency to any endpoint that accesses channel-specific data."""
    channel = await get_channel(session, channel_id)
    if not channel:
        raise HTTPException(404, "Channel not found")
    if channel.author_id != current_author.id:
        raise HTTPException(403, "not_channel_owner")
    return channel

# Usage:
@router.get("/{channel_id}/analytics")
async def get_analytics(
    channel: Channel = Depends(require_channel_owner),
) -> AnalyticsResponse:
    ...
```

## Pattern 4: Telegram initData Verification (TWA)

```python
import hmac
import hashlib
import time
from urllib.parse import parse_qs

def verify_telegram_init_data(init_data: str) -> dict:
    """Verify Telegram Web App initData. Must be checked before issuing JWT."""
    bot_token = read_secret("telegram_bot_token")
    parsed = parse_qs(init_data)
    
    if "hash" not in parsed:
        raise ValueError("Missing hash")
    
    received_hash = parsed.pop("hash")[0]
    
    # Check expiry (auth_date must be < 1 hour old)
    auth_date = int(parsed.get("auth_date", ["0"])[0])
    if time.time() - auth_date > 3600:
        raise ValueError("initData expired")
    
    # Verify HMAC
    data_check = "\n".join(
        f"{k}={v[0]}" for k, v in sorted(parsed.items())
    )
    secret_key = hmac.new(
        b"WebAppData",
        bot_token.encode("utf-8"),
        hashlib.sha256,
    ).digest()
    expected_hash = hmac.new(
        secret_key,
        data_check.encode("utf-8"),
        hashlib.sha256,
    ).hexdigest()
    
    if not hmac.compare_digest(expected_hash, received_hash):
        raise ValueError("Invalid hash")
    
    return {k: v[0] for k, v in parsed.items()}
```

## Pattern 5: Telegram Bot Webhook Verification

```python
from aiogram import Bot

async def verify_telegram_secret_token(
    request: Request,
    bot: Bot = Depends(get_bot),
) -> None:
    """Verify Telegram webhook secret token."""
    token = request.headers.get("X-Telegram-Bot-Api-Secret-Token")
    expected = read_secret("telegram_webhook_secret")
    if not token or not hmac.compare_digest(token, expected):
        raise HTTPException(401, "Unauthorized")
```

## Pattern 6: Payment Idempotency Check

```python
async def check_payment_idempotency(
    session: AsyncSession,
    yukassa_payment_id: str,
) -> bool:
    """Returns True if payment already processed (safe to skip)."""
    existing = await session.scalar(
        select(Payment).where(
            Payment.yukassa_payment_id == yukassa_payment_id,
            Payment.status == "success",
        )
    )
    return existing is not None

# Usage in webhook handler:
if await check_payment_idempotency(session, payment_id):
    return {"status": "ok", "message": "already processed"}
```

## Pattern 7: Rate Limiting (Redis)

```python
import aioredis
from datetime import datetime

async def check_rate_limit(
    redis: aioredis.Redis,
    user_id: int,
    limit: int = 30,
    window_seconds: int = 60,
) -> bool:
    """Sliding window rate limit. Returns False if limit exceeded."""
    key = f"rate_limit:bot:{user_id}"
    now = datetime.utcnow().timestamp()
    window_start = now - window_seconds
    
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)
    pipe.zadd(key, {str(now): now})
    pipe.zcard(key)
    pipe.expire(key, window_seconds)
    results = await pipe.execute()
    
    count = results[2]
    return count <= limit
```

## Security invariants (never violate)

1. HMAC check is ALWAYS the first operation in any webhook handler
2. JWT is verified + author extracted BEFORE channel ownership check
3. Channel ownership ALWAYS verified before returning channel-specific data
4. idempotency_key is ALWAYS a new `uuid.uuid4()` per payment attempt
5. Duplicate webhook detection ALWAYS happens before DB writes
6. Secrets NEVER appear in logs (only IDs, never values)
