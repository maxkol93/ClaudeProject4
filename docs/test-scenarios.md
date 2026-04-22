# BDD Test Scenarios: TelePub
**Дата:** 2026-04-22 | **Генерация:** requirements-validator

---

## Feature: Author Registration (US-01)

```gherkin
Feature: Author Channel Registration

  Background:
    Given TelePubBot is running and accessible
    And ЮKassa OAuth endpoint is available

  # Happy Path
  Scenario: Successful author registration within 5 minutes
    Given user @maxkol93 is the admin of Telegram channel @finance_pro
    When he sends /register to @TelePubBot
    And follows the channel connection dialog
    And adds @TelePubBot as admin with Invite + Ban permissions
    And sets subscription price to 299 RUB/month
    And completes ЮKassa OAuth authorization
    Then he receives a subscribe link: t.me/TelePubBot?start=channel_{id}
    And channel status in dashboard is "Active"
    And the entire process takes less than 5 minutes

  Scenario: Author with multiple channels registers second channel
    Given @maxkol93 already has channel @finance_pro connected
    When he sends /add_channel and connects @crypto_insider
    Then both channels appear in his dashboard
    And each has its own subscribe link and price settings

  # Error Handling
  Scenario: Bot not added as admin
    Given @maxkol93 added @TelePubBot to channel without admin rights
    When he clicks "Проверить ✅" button
    Then bot sends message: "❌ Бот не добавлен как администратор"
    And lists required permissions: Invite Users, Ban Users
    And registration pauses at "awaiting_bot_admin" state

  Scenario: User is not channel admin
    Given @maxkol93 is a regular member (not admin) of @someone_channel
    When he sends /register and inputs @someone_channel
    Then bot sends: "❌ Вы не являетесь администратором этого канала"
    And registration does not proceed

  Scenario: ЮKassa OAuth failure
    Given ЮKassa OAuth service returns an error
    When @maxkol93 clicks "Подключить ЮKassa"
    And authorization fails
    Then bot sends: "❌ Не удалось подключить ЮKassa. Попробуйте позже."
    And channel is saved but marked as "payment_pending" (not active)
    And user can retry OAuth without restarting registration

  # Edge Cases
  Scenario: Price below minimum
    Given user is at "set price" step
    When he enters "50" (below 100 RUB minimum)
    Then bot responds: "❌ Минимальная цена — 100 руб."
    And asks to enter price again

  Scenario: Non-existent channel username
    Given user sends "@nonexistent_channel_xyz" to bot
    When bot tries to fetch channel info
    Then bot responds: "❌ Канал не найден. Проверьте username и попробуйте ещё раз."

  # Security
  Scenario: CSRF attempt on ЮKassa OAuth callback
    Given an attacker intercepts OAuth callback URL
    When they replay the callback with different state parameter
    Then server rejects with 400 "Invalid OAuth state"
    And no ЮKassa credentials are saved

  Scenario: Bot token hijacking attempt
    Given an attacker sends fake Telegram webhook without valid secret
    When POST /telegram/webhook arrives without X-Telegram-Bot-Api-Secret-Token
    Then server responds 401 Unauthorized
    And incident is logged with attacker IP
```

---

## Feature: Subscription Purchase (US-02)

```gherkin
Feature: Reader Subscription Flow

  Background:
    Given channel @finance_pro is connected to TelePub
    And subscription price is 299 RUB/month
    And ЮKassa is operational

  # Happy Path
  Scenario: Successful subscription via card payment
    Given reader @reader_alex opens t.me/TelePubBot?start=channel_{id}
    When bot shows channel info and "Подписаться за 299 руб/мес" button
    And @reader_alex clicks the button and selects "Банковская карта"
    And completes payment on ЮKassa payment page
    Then ЮKassa sends webhook with status "succeeded"
    And within 30 seconds @reader_alex is added to private channel
    And receives welcome message: "✅ Добро пожаловать в @finance_pro!"
    And author's dashboard shows +1 subscriber and +299 RUB revenue

  Scenario: Successful subscription via Telegram Stars
    Given @reader_alex chooses "Telegram Stars" payment option
    When he completes Stars payment in Telegram interface
    Then Telegram sends payment webhook
    And within 30 seconds access is granted
    And Stars are deducted from his Telegram account

  Scenario: Free trial activation
    Given channel offers 7-day free trial
    When @reader_alex clicks "Попробовать бесплатно (7 дней)"
    Then he is added to channel immediately without payment
    And subscription status is "trial"
    And expires_at = NOW() + 7 days
    And 3 days before trial ends he receives renewal notification

  # Error Handling
  Scenario: Card payment declined by bank
    Given @reader_alex enters valid card format but bank declines
    When ЮKassa webhook arrives with status "canceled"
    Then bot sends: "❌ Платёж отклонён. Попробуйте другую карту или способ оплаты."
    And access to channel is NOT granted
    And no payment record with status "success" is created

  Scenario: Payment webhook timeout (arrives 10 minutes late)
    Given @reader_alex completes payment but webhook is delayed
    When reconciliation job runs after 10 minutes
    Then it queries ЮKassa API directly for payment status
    And if status is "succeeded" — grants access retroactively
    And logs "late_webhook_recovery" event

  Scenario: User tries to subscribe when already subscribed
    Given @reader_alex has active subscription to @finance_pro
    When he opens subscribe link again
    Then bot shows: "✅ У вас уже есть активная подписка до [date]"
    And shows "Управление подпиской" options
    And does NOT create duplicate subscription

  # Edge Cases
  Scenario: Simultaneous payment processing (double-click)
    Given @reader_alex clicks "Pay" twice rapidly
    When two payment requests are created with same idempotency key
    Then ЮKassa deduplicates — only one charge occurs
    And platform processes only one webhook (idempotency check)
    And subscriber receives only one confirmation message

  # Security
  Scenario: Forged ЮKassa webhook (HMAC manipulation)
    Given attacker sends POST /webhooks/yukassa with fake payload
    When HMAC-SHA256 signature does not match YUKASSA_SECRET
    Then server returns 400 "Invalid signature"
    And no subscription is created
    And incident logged with IP and payload hash

  Scenario: Webhook replay attack
    Given payment "pay_abc123" was processed successfully
    When same webhook payload is replayed by attacker
    Then idempotency check finds existing payment with status "success"
    And returns 200 OK without processing again
    And no duplicate access grant or balance credit
```

---

## Feature: Subscription Lifecycle (US-03)

```gherkin
Feature: Automatic Access Management

  # Happy Path
  Scenario: Successful monthly renewal
    Given @reader_alex has active subscription expiring in 1 day
    And yukassa_payment_method_id is saved from initial payment
    When daily renewal job runs at 09:00 MSK
    Then platform charges saved payment method for 299 RUB
    And subscription expires_at extends by 30 days
    And @reader_alex remains in channel uninterrupted
    And receives notification: "✅ Подписка продлена до [new_date]"

  Scenario: Subscription cancelled by user
    Given @reader_alex has active subscription
    When he sends /cancel to bot and confirms cancellation
    Then subscription status changes to "cancelled"
    And @reader_alex is immediately removed from private channel
    And receives: "Ваша подписка отменена. Жаль расставаться! 👋"
    And author sees -1 subscriber in dashboard

  # Error Handling
  Scenario: Renewal failed → grace period → expired
    Given @reader_alex's subscription expires today
    And his saved card is declined during auto-renewal
    When daily renewal job processes his subscription
    Then status changes to "grace"
    And @reader_alex receives: "❌ Не удалось списать 299 руб. Обновите способ оплаты."
    And he remains in channel for 24 hours (grace period)
    After 24 hours without payment:
    Then status changes to "expired"
    And @reader_alex is removed from channel
    And receives: "⏰ Доступ закрыт. Подпишитесь снова по ссылке: [link]"

  # Edge Cases
  Scenario: Race condition — renewal and manual cancel simultaneously
    Given @reader_alex's subscription is being renewed
    When he simultaneously clicks "Cancel" in dashboard
    Then database pessimistic lock prevents inconsistent state
    And either renewal succeeds (cancel ignored) OR cancel succeeds (renewal refunded)
    And exactly one final state is persisted

  Scenario: Bot kicked from channel by author
    Given author accidentally removes @TelePubBot from @finance_pro
    When bot tries to grant access to new subscriber
    Then Telegram returns ChatAdminRequired error
    And author receives: "⚠️ Бот удалён из канала! Добавьте обратно для продолжения работы."
    And new subscriptions are paused but existing continue
```

---

## Feature: Author Payouts (US-04)

```gherkin
Feature: Author Earnings Payouts

  Background:
    Given author @maxkol93 has verified payout card linked
    And has balance 5,000 RUB

  # Happy Path
  Scenario: Successful automatic daily payout
    Given today is a regular business day
    When Celery Beat triggers payout job at 10:00 MSK
    Then ЮKassa payout is initiated for 5,000 RUB
    And author's balance resets to 0
    And payout record created with status "processing"
    And @maxkol93 receives: "💸 Выплата 5,000 руб инициирована. Ожидайте 1-24 часа."
    When ЮKassa confirms payout completion
    Then payout status updates to "completed"
    And @maxkol93 receives: "✅ Выплата 5,000 руб зачислена на карту!"

  Scenario: Manual payout request
    Given balance is 3,500 RUB (above 1,000 minimum)
    When @maxkol93 sends /payout in bot
    Then system initiates payout immediately (outside daily schedule)
    And responds with payout confirmation

  # Error Handling
  Scenario: Balance below minimum threshold
    Given balance is 750 RUB
    When daily payout job runs
    Then no payout is initiated
    And balance remains 750 RUB
    And no notification sent (silent accumulation)
    When balance reaches 1,000 RUB on a later day
    Then payout is initiated automatically

  Scenario: ЮKassa payout service unavailable
    Given ЮKassa returns 503 on payout request
    When payout job attempts payout
    Then payout record created with status "failed"
    And balance is NOT decremented (rollback)
    And @maxkol93 receives: "⚠️ Выплата временно недоступна. Попробуем снова завтра."
    And retry scheduled for next daily run

  # Edge Cases
  Scenario: Large payout exceeding ЮKassa limit (600K RUB)
    Given author has balance 750,000 RUB
    When payout job runs
    Then payout is split: 600,000 RUB first, 150,000 RUB second
    And two separate ЮKassa payout requests created
    And author receives single notification with total amount

  # Security
  Scenario: Attempt to trigger payout for another author
    Given @attacker_user is authenticated as author A
    When they POST /api/channels/author_B_channel/payouts/manual
    Then server returns 403 Forbidden "not_channel_owner"
    And no payout is initiated for author B
```

---

## Feature: Analytics Dashboard (US-05)

```gherkin
Feature: Channel Analytics

  # Happy Path
  Scenario: Author views 30-day analytics
    Given channel @finance_pro has 150 active subscribers
    When @maxkol93 opens Analytics in dashboard
    Then he sees:
      | Metric              | Value  |
      | MRR                 | 44,850 руб |
      | Active subscribers  | 150    |
      | New this month      | 23     |
      | Churned this month  | 8      |
      | ARPU                | 299 руб |
      | LTV estimate        | ~5,980 руб |
    And data was last updated less than 1 hour ago
    And "Export CSV" button is available

  Scenario: Export analytics to CSV
    Given @maxkol93 is viewing analytics for period "last 30 days"
    When he clicks "Экспорт CSV"
    Then CSV file downloads within 5 seconds
    And contains all metrics with daily breakdown
    And file is named "finance_pro_analytics_2026-04.csv"

  # Error Handling
  Scenario: Analytics for new channel with 0 subscribers
    Given @maxkol93 just registered channel with 0 subscribers
    When he opens Analytics
    Then dashboard shows all metrics as 0 or "—"
    And displays helpful message: "Поделитесь ссылкой на подписку чтобы начать!"
    And does NOT show division-by-zero errors

  Scenario: Analytics cache miss (Redis down)
    Given Redis is temporarily unavailable
    When @maxkol93 requests analytics
    Then system falls back to direct PostgreSQL query
    And displays data (may be slightly slower)
    And does NOT return 500 error

  # Edge Cases
  Scenario: Period with no activity
    Given no new subscriptions or cancellations in selected period
    When analytics are calculated
    Then new=0, churned=0, MRR trend=0%
    And system shows "Нет данных за период" for trend chart (not error)
```

---

## Security Scenarios (Cross-Feature)

```gherkin
Feature: Platform Security

  Scenario: SQL injection via bot message
    Given attacker sends message: "'; DROP TABLE subscriptions; --"
    When bot processes the message as channel username input
    Then SQLAlchemy ORM parameterizes the query
    And no SQL is executed from user input
    And bot responds with normal "channel not found" message

  Scenario: JWT token from another user
    Given @attacker has valid JWT for their own account
    When they use that JWT to request analytics for @maxkol93's channel
    Then server decodes JWT, extracts attacker's telegram_user_id
    And verifies attacker does NOT own the requested channel
    And returns 403 Forbidden

  Scenario: Rate limit exceeded on bot
    Given user sends 35 messages in 1 minute to bot (limit: 30/min)
    When 31st message arrives
    Then bot temporarily ignores messages from this user
    And after 60 seconds, normal operation resumes
    And excessive requests are logged

  Scenario: Telegram initData tampering for TWA
    Given attacker modifies initData hash in Telegram Web App
    When POST /api/auth/twa-login with tampered initData
    Then HMAC verification fails
    And server returns 401 Unauthorized
    And no JWT is issued
```
