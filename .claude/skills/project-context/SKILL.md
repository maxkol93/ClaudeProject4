---
name: project-context
description: TelePub domain knowledge — Telegram channel monetization platform for RU/CIS market. Contains payment provider specifics (ЮKassa), Telegram API quirks, business rules, and competitive context. Load when implementing any feature to ground decisions in product requirements.
version: "1.0"
maturity: production
---

# TelePub — Project Context

## Product

**TelePub** — Telegram-native channel monetization platform for Russian market.

Authors connect their Telegram channels and accept paid subscriptions from readers. Platform handles payment processing (ЮKassa), automatic access management, renewals, and payouts.

**Competitors:** Boosty (10-20% fee, complex), Tribute (5% fee, minimal features)
**Differentiator:** 0→5% freemium fee, 24-hour payouts, Telegram-first UX

## Business rules

### Commission model
```
author_monthly_revenue ≤ 10,000 RUB → platform_fee = 0%
author_monthly_revenue > 10,000 RUB → platform_fee = 5%

net_to_author = payment_amount - platform_fee - yukassa_fee
```

### Payout rules
- Minimum balance: 1,000 RUB
- Auto-payout daily at 10:00 MSK
- Manual payout via /payout bot command
- Target delivery: 24 hours (ЮKassa payout API)
- Large payouts (>600K RUB): split into multiple requests

### Subscription lifecycle
```
pending → active (after payment)
active → grace (renewal failed)
grace → expired (24h without payment)
active → cancelled (user request, immediate revocation)
expired → active (re-subscribe = new payment)
```

### Access management timing
- Grant access: within 30 seconds of payment confirmation
- Revoke access: immediately on cancellation/expiry
- Renewal window: attempt renewal when expires_at < NOW() + 3 days

## ЮKassa integration

### Authentication
```python
from yookassa import Configuration
Configuration.account_id = read_secret("yukassa_shop_id")
Configuration.secret_key = read_secret("yukassa_secret_key")
```

### Creating a payment
```python
from yookassa import Payment
import uuid

payment = Payment.create({
    "amount": {"value": "299.00", "currency": "RUB"},
    "confirmation": {"type": "redirect", "return_url": "https://t.me/TelePubBot"},
    "capture": True,
    "payment_method_data": {"type": "bank_card"},
    "metadata": {"subscription_id": str(subscription.id)},
    "description": f"Подписка на {channel.title}",
}, idempotency_key=str(uuid.uuid4()))
```

### Webhook verification
```python
import hmac, hashlib

def verify_yukassa_hmac(payload_bytes: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), payload_bytes, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)
```

### Rebilling (recurring)
```python
payment = Payment.create({
    "amount": {"value": "299.00", "currency": "RUB"},
    "capture": True,
    "payment_method_id": subscription.yukassa_payment_method_id,  # saved from initial
    "metadata": {"subscription_id": str(subscription.id), "renewal": True},
}, idempotency_key=str(uuid.uuid4()))
```

## Telegram Bot API quirks

### Removing channel member (critical gotcha)
```python
# WRONG — permanent ban, user can never re-subscribe
await bot.ban_chat_member(channel_id, user_id)

# CORRECT — remove without blocking re-subscribe
await bot.ban_chat_member(channel_id, user_id)
await bot.unban_chat_member(channel_id, user_id)  # must follow immediately
```

### Single-use invite links
```python
link = await bot.create_chat_invite_link(
    chat_id=channel_id,
    member_limit=1,  # expires after one use
    name=f"sub_{subscription_id}",
)
```

### Rate limits
- 30 messages/second per bot globally
- 1 message/second per user
- For bulk notifications: use aiogram throttling middleware

### Bot admin permissions required
- `can_invite_users=True` — for generating invite links
- `can_restrict_members=True` — for ban/unban operations

## Telegram Web App (TWA)

### initData verification
```python
import hmac, hashlib
from urllib.parse import parse_qs

def verify_telegram_init_data(init_data: str, bot_token: str) -> dict:
    parsed = parse_qs(init_data)
    received_hash = parsed.pop("hash")[0]
    data_check = "\n".join(f"{k}={v[0]}" for k, v in sorted(parsed.items()))
    secret_key = hmac.new(b"WebAppData", bot_token.encode(), hashlib.sha256).digest()
    expected_hash = hmac.new(secret_key, data_check.encode(), hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected_hash, received_hash):
        raise ValueError("Invalid initData")
    if time.time() - int(parsed["auth_date"][0]) > 3600:
        raise ValueError("initData expired")
    return {k: v[0] for k, v in parsed.items()}
```

## Market context

- **Russia/CIS only** — ЮKassa handles 152-ФЗ compliance
- **Telegram penetration**: 76M+ monthly active users in Russia
- **Target channels**: 200K+ channels with 1K+ subscribers
- **Typical price**: 99-999 RUB/month per channel

## Key personas

| Persona | Need | Pain point |
|---------|------|-----------|
| Author | Monetize Telegram channel | Complex setup, high fees, slow payouts |
| Subscriber | Support favorite channels | Fragmented payment flows |

## MVP feature set (US-01 through US-05)

| Story | Feature | Status |
|-------|---------|--------|
| US-01 | Author registration (FSM) | Planned |
| US-02 | Subscriber payment flow | Planned |
| US-03 | Auto access management (lifecycle) | Planned |
| US-04 | Author payouts | Planned |
| US-05 | Analytics dashboard | Planned |
