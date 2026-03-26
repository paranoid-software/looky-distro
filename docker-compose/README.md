# Docker Compose deployment

Deploy looky using Docker Compose (single host).

## Quick start

From the repo root:

```bash
docker login ghcr.io -u USERNAME --password-stdin <<< "$GITHUB_TOKEN"
cp docker-compose/.env.example docker-compose/.env
# Edit docker-compose/.env with your values (see "Generating keys" below)
docker compose --env-file docker-compose/.env -f docker-compose/docker-compose.yml up -d
```

**Note:** You do not need to pre-create the Postgres data or workspaces directories. Docker creates them automatically when using bind mounts if they do not exist.

## Generating keys

Fill these in `.env` before first run:

**LOOKY_SESSION_SECRET** (random string for session signing):

```bash
openssl rand -hex 32
```

**EXPORT_ENGINE_SHARED_TOKEN** (shared secret between `app` and `export-engine` for scheduled PDF renders):

```bash
openssl rand -hex 32
```

**INVITATION_TOKEN_ENCRYPTION_KEY** (Fernet key for invitation tokens):

```bash
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

**STYTCH_*** — Get from [Stytch Dashboard](https://dashboard.stytch.com/) (Project ID, Secret, etc.).

**STATIC_ASSET_VERSION** should be set to the asset version shipped with the release. In the standard image flow this should normally match `IMAGE_TAG`.

## Ports

- App: 8000
- Postgres: 55432
- Malloy service: 3002
- Redis: 6379
