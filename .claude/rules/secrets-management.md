# Secrets Management: TelePub

## Principle

**Zero secrets in code or environment variables in docker-compose.yml.**
All secrets via Docker secrets mounted at `/run/secrets/`.

## Secret inventory

| Secret | Path | Used by |
|--------|------|---------|
| Telegram Bot Token | `/run/secrets/telegram_bot_token` | bot service |
| ЮKassa Shop Secret | `/run/secrets/yukassa_secret_key` | api + worker |
| ЮKassa Shop ID | `/run/secrets/yukassa_shop_id` | api + worker |
| JWT Secret | `/run/secrets/jwt_secret` | api |
| PostgreSQL password | `/run/secrets/postgres_password` | all services |
| Redis password | `/run/secrets/redis_password` | all services |

## Reading secrets in Python

```python
from pathlib import Path

def read_secret(name: str) -> str:
    secret_path = Path(f"/run/secrets/{name}")
    if secret_path.exists():
        return secret_path.read_text().strip()
    # Fallback for local dev only
    import os
    return os.environ[name.upper()]
```

## docker-compose.yml pattern

```yaml
secrets:
  telegram_bot_token:
    file: ./secrets/telegram_bot_token.txt
  yukassa_secret_key:
    file: ./secrets/yukassa_secret_key.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt

services:
  bot:
    secrets:
      - telegram_bot_token
      - yukassa_secret_key
```

## Local development

```bash
# Create secrets directory (gitignored)
mkdir -p secrets/
echo "your_bot_token_here" > secrets/telegram_bot_token.txt
echo "your_yukassa_secret_here" > secrets/yukassa_secret_key.txt

# Verify .gitignore covers it
grep "secrets/" .gitignore || echo "secrets/" >> .gitignore
```

## .env.example (safe to commit — placeholder values only)

```bash
TELEGRAM_BOT_TOKEN=your_token_here
YUKASSA_SHOP_ID=your_shop_id_here
YUKASSA_SECRET_KEY=your_secret_here
JWT_SECRET=generate_32_random_bytes_here
POSTGRES_PASSWORD=change_this
REDIS_PASSWORD=change_this
```

## Rotation procedure

1. Generate new secret value
2. Update Docker secret file on VPS: `echo "new_value" > /run/secrets/secret_name`
3. Restart affected service: `docker compose restart [service]`
4. Verify with health check: `curl https://telepub.ru/api/health`

## What NEVER goes in git

- `secrets/` directory
- `.env` with real values
- Any file matching `*_secret*`, `*_token*`, `*_key*` (except `.example`)

## CI/CD (GitHub Actions)

Secrets stored in GitHub repository secrets, injected as env vars during deploy job.
Never printed in logs (`::add-mask::$SECRET_VALUE` in workflow if needed).
