# looky — Production images

Config-driven analytics platform. Images are published to GitHub Container Registry (GHCR).

## Access

Images are private. **Contact us to obtain pull access** — [your contact info here].

## Images

| Image | Purpose |
|-------|---------|
| `ghcr.io/paranoid-software/looky/app` | Flask API + UI |
| `ghcr.io/paranoid-software/looky/query-engine` | Malloy runtime |
| `ghcr.io/paranoid-software/looky/export-engine` | PDF export (scheduler + worker) |
| `ghcr.io/paranoid-software/looky/schema` | DB init + migrations |

Tag format: `yyyymmdd.run_number` (e.g. `20260317.1`). Set `IMAGE_TAG` in your deployment config.

## Deployment guides

| Deployment | Description |
|------------|-------------|
| [Docker Compose](docker-compose/) | Single-host deployment with Docker Compose |
| [Kubernetes (GKE)](kubernetes/) | Google Kubernetes Engine *(coming soon)* |

Each guide includes its own compose/manifests and env examples.
