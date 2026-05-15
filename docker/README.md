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

**Note:** `POSTGRES_DATA_PATH` and `WORKSPACES_PATH` can be left to Docker — it creates the directories on first start when using bind mounts.

`VAULT_DATA_PATH` is the exception: the `secrets-vault` container is distroless and runs as uid `65532`, so it cannot `chown` its own mount. If Docker auto-creates the directory it ends up owned by `root:root`, and every settings PUT will 500 with `EACCES: permission denied, mkdir '/vault-data/...'`. Pre-create it with the right owner before the first `up`:

```bash
mkdir -p "$VAULT_DATA_PATH"
sudo chown 65532:65532 "$VAULT_DATA_PATH"
```

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
