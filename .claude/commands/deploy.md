# /deploy — Deploy to VPS

Deploy TelePub to production VPS (AdminVPS/HostKey) via Docker Compose.

## Usage

```
/deploy                    # Full deploy (build + restart all services)
/deploy --service=bot      # Restart single service
/deploy --migrate          # Run DB migrations only
/deploy --rollback         # Rollback to previous image
/deploy --check            # Check deployment status
```

## Pre-deploy checklist

Before running any deploy:
- [ ] Tests pass: `pytest tests/unit/ -x -q`
- [ ] Type check passes: `mypy bot/ api/ worker/ shared/ --strict --ignore-missing-imports`
- [ ] Linting passes: `ruff check .`
- [ ] No secrets in diff: `git diff HEAD~1 | grep -iE "(token|secret|password)"`
- [ ] Committed and pushed to main

## Deploy sequence

```bash
# On VPS (via SSH)
ssh user@vps.hostname

cd /opt/telepub

# 1. Pull latest
git pull origin main

# 2. Build images
docker compose build --no-cache

# 3. Run DB migrations
docker compose run --rm api alembic upgrade head

# 4. Rolling restart (bot first, then api, worker)
docker compose up -d --no-deps bot
docker compose up -d --no-deps api
docker compose up -d --no-deps worker beat

# 5. Verify health
curl https://telepub.ru/api/health
docker compose ps
```

## Rollback

```bash
# If deploy fails
docker compose stop bot api worker beat
docker compose up -d --no-deps bot api worker beat  # uses previous image if build failed
# OR
git revert HEAD
git push origin main
# then redeploy
```

## DB rollback (Alembic)

```bash
docker compose run --rm api alembic downgrade -1
```

## Health check

```bash
# Service status
docker compose ps

# Logs
docker compose logs bot --tail=50
docker compose logs api --tail=50
docker compose logs worker --tail=50

# Health endpoint
curl -f https://telepub.ru/api/health || echo "UNHEALTHY"

# Celery workers
docker compose exec worker celery inspect ping
```

## Environment: VPS

- VPS provider: AdminVPS/HostKey
- Domain: telepub.ru
- Deploy path: `/opt/telepub/`
- Docker Compose file: `docker-compose.yml`
- Secrets path: `/run/secrets/`
- SSL: Let's Encrypt via Nginx

## Monitoring post-deploy

Check Grafana dashboard for:
- Bot response latency (should be <500ms p95)
- Payment webhook success rate (should be >99%)
- Celery task queue depth (should stay <100)
- PostgreSQL connections (should stay <50)
