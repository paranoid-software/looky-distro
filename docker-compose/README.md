# Docker Compose deployment

Deploy looky using Docker Compose (single host).

## Quick start

From the repo root:

```bash
docker login ghcr.io -u USERNAME --password-stdin <<< "$GITHUB_TOKEN"
cp docker-compose/.env.example docker-compose/.env
# Edit docker-compose/.env with your values
docker compose --env-file docker-compose/.env -f docker-compose/docker-compose.yml up -d
```

## Ports

- App: 8000
- Postgres: 55432
- Malloy service: 3002
- Redis: 6379
