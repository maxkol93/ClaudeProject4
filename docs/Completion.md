# Completion: TelePub
**Дата:** 2026-04-22

---

## 1. Pre-Deployment Checklist

### Development
- [ ] Все unit tests проходят (`pytest --cov=80`)
- [ ] Все integration tests проходят
- [ ] Linting без ошибок (`ruff check .`)
- [ ] Type checking (`mypy .`)
- [ ] Алembic миграции применены и проверены
- [ ] `.env.example` обновлён

### Security
- [ ] ЮKassa webhook HMAC верификация — протестирована с реальным webhook
- [ ] JWT secret сгенерирован (32+ байт случайных)
- [ ] Bot token НЕ в репозитории (только в Docker secrets)
- [ ] PostgreSQL не exposed на внешний интерфейс
- [ ] Redis AUTH пароль установлен
- [ ] Nginx rate limiting настроен
- [ ] Let's Encrypt SSL получен для telepub.ru

### Infrastructure
- [ ] VPS (AdminVPS/HostKey) заказан и доступен
- [ ] DNS telepub.ru → VPS IP настроен
- [ ] Docker и Docker Compose установлены на VPS
- [ ] PostgreSQL данные на /data/postgres (volume)
- [ ] Redis данные на /data/redis (volume)
- [ ] Backup cron настроен

### Telegram
- [ ] @TelePubBot создан через @BotFather
- [ ] Bot token получен и сохранён в secrets/
- [ ] Webhook URL установлен: https://telepub.ru/telegram/webhook
- [ ] Webhook secret token установлен

### ЮKassa
- [ ] Аккаунт продавца создан и верифицирован
- [ ] Shop ID и Secret Key получены
- [ ] Webhook URL зарегистрирован: https://telepub.ru/webhooks/yukassa
- [ ] Тестовые платежи прошли успешно

---

## 2. Deployment Sequence

```bash
# 1. На VPS: клонируем репозиторий
git clone https://github.com/your-org/telepub.git
cd telepub

# 2. Создаём директории для данных и secrets
mkdir -p /data/postgres /data/redis /data/minio secrets

# 3. Заполняем secrets (никогда не в git!)
echo "your_bot_token" > secrets/telegram_bot_token.txt
echo "your_yukassa_secret" > secrets/yukassa_secret.txt
echo "strong_db_password" > secrets/db_password.txt
echo "$(openssl rand -hex 32)" > secrets/jwt_secret.txt
chmod 600 secrets/*

# 4. Копируем конфиг
cp .env.example .env
# Заполняем .env (не секреты — только non-sensitive конфиг)

# 5. SSL сертификат (до запуска nginx)
docker run --rm -v /data/certbot:/etc/letsencrypt certbot/certbot \
  certonly --standalone -d telepub.ru --email admin@telepub.ru --agree-tos

# 6. Собираем и запускаем
docker compose pull
docker compose build
docker compose up -d postgres redis  # Сначала данные
sleep 10

# 7. Применяем миграции
docker compose run --rm api alembic upgrade head

# 8. Запускаем все сервисы
docker compose up -d

# 9. Устанавливаем Telegram webhook
docker compose run --rm bot python -m scripts.set_webhook

# 10. Проверяем здоровье
docker compose ps
curl https://telepub.ru/api/health
```

---

## 3. Rollback Procedure

```bash
# Быстрый rollback к предыдущей версии
git log --oneline -5  # Найти предыдущий commit

# Вариант A: rollback Docker image
docker compose down api bot worker
git checkout <previous_commit>
docker compose build api bot worker
docker compose up -d api bot worker

# Вариант B: rollback миграции (если применялись)
docker compose run --rm api alembic downgrade -1

# Проверка
docker compose logs -f api bot worker
curl https://telepub.ru/api/health
```

---

## 4. CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy TelePub

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
      redis:
        image: redis:7
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt -r requirements-dev.txt
      - run: ruff check .
      - run: mypy .
      - run: pytest --cov=80

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to VPS
        env:
          VPS_HOST: ${{ secrets.VPS_HOST }}
          VPS_USER: ${{ secrets.VPS_USER }}
          VPS_KEY: ${{ secrets.VPS_PRIVATE_KEY }}
        run: |
          ssh -i "$VPS_KEY" "$VPS_USER@$VPS_HOST" '
            cd /srv/telepub &&
            git pull origin main &&
            docker compose build &&
            docker compose run --rm api alembic upgrade head &&
            docker compose up -d --no-deps api bot worker beat
          '
```

---

## 5. Monitoring & Alerting

### Key Metrics (Prometheus + Grafana)

| Метрика | Threshold | Alert channel |
|---------|-----------|---------------|
| Bot webhook p95 latency | > 500ms | Telegram admin chat |
| Payment webhook processing rate | < 0.95 success | Telegram admin chat |
| PostgreSQL connections | > 80% pool | Email |
| Redis memory usage | > 80% | Email |
| Celery queue length (critical) | > 100 tasks | Telegram admin chat |
| Error rate (5xx) | > 1% | Telegram admin chat |
| Bot downtime | > 1 min | Telegram admin chat |
| Disk usage | > 80% | Email |

### Dashboards (Grafana)

1. **Business Dashboard:** MRR, новые подписчики, выплаты, churn
2. **Technical Dashboard:** Response times, error rates, queue lengths
3. **Payment Dashboard:** Успешность платежей, webhook processing time

### Logging (structlog + Loki)

```python
# Пример structured log
import structlog
log = structlog.get_logger()

log.info("payment_processed",
    payment_id=payment.id,
    subscription_id=payment.subscription_id,
    amount_rub=payment.amount_rub,
    status="success"
)
```

### Health Check Endpoint

```
GET /api/health
Response: {
  "status": "ok",
  "db": "ok",
  "redis": "ok",
  "celery": "ok",
  "version": "1.0.0"
}
```

---

## 6. Logging Strategy

| Level | Что логируем |
|-------|-------------|
| INFO | Платежи (без PII), подписки created/expired, выплаты |
| WARNING | Retry события, degraded service, telegram API errors |
| ERROR | Payment failures, webhook HMAC mismatch, DB errors |
| CRITICAL | Service down, data integrity issues |

**Retention:** 30 дней (INFO), 90 дней (WARNING+), 1 год (платёжные события)

**PII:** telegram_user_id логируется, имена/карты — НИКОГДА

---

## 7. Handoff Checklists

### For Development Team
- [ ] README.md с инструкцией запуска за < 5 минут
- [ ] DEVELOPMENT_GUIDE.md с описанием архитектуры
- [ ] Доступ к репозиторию и VPS
- [ ] `.env.example` с описанием каждой переменной
- [ ] Telegram test bot для dev окружения

### For QA Team
- [ ] Staging окружение с тестовыми ЮKassa credentials
- [ ] Набор тестовых Telegram аккаунтов
- [ ] Test plan для payment flows
- [ ] Доступ к Grafana dashboard

### For Operations
- [ ] VPS credentials в password manager
- [ ] Runbook: как перезапустить сервисы
- [ ] Runbook: как применить emergency hotfix
- [ ] Runbook: что делать если ЮKassa недоступна
- [ ] Escalation: Telegram admin chat для алертов
- [ ] Backup verification procedure (раз в неделю тестовый restore)

---

## 8. Post-Launch Verification (Day 1 Checklist)

- [ ] Бот отвечает на /start
- [ ] Тестовая регистрация автора прошла
- [ ] Тестовый платёж через ЮKassa (тестовая карта) обработан
- [ ] Доступ к каналу выдан автоматически
- [ ] Ежедневный payout job запустился
- [ ] Grafana dashboard показывает данные
- [ ] Логи без CRITICAL/ERROR
- [ ] Backup завершился успешно
