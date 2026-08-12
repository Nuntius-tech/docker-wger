# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Docker Compose stack configurations for [wger](https://github.com/wger-project/wger), a self-hosted workout manager. Three deployable environments live here; the application code itself is in a separate repo.

## Environments

| Directory | Database | Use case |
|-----------|----------|----------|
| `.` (root) | PostgreSQL | Production |
| `dev/` | SQLite | Development (requires `WGER_CODEPATH` pointing to the wger source checkout) |
| `dev-postgres/` | PostgreSQL | Development with Postgres (requires `WGER_CODEPATH`) |

## Common Commands

```bash
# Production
docker compose up -d
docker compose exec web ./manage.py setup-powersync-storage

# Development (SQLite) — must set WGER_CODEPATH first
cd dev
docker compose up -d
docker compose watch   # live-reload on code changes

# Development (Postgres)
cd dev-postgres
docker compose up -d

# Sync exercises manually
docker compose exec web python3 manage.py sync-exercises
docker compose exec web python3 manage.py download-exercise-images

# Generate fresh JWT keys
docker compose exec web ./manage.py generate-jwt-keys

# Run nginx tests
cd tests
docker compose -f docker-compose.test.yml up --abort-on-container-exit test
docker compose -f docker-compose.test.yml down -v   # cleanup
```

## Architecture

```
nginx (port 80)
  └── web (wger Django app, port 8000)
        ├── db (postgres:15, logical WAL replication enabled)
        ├── cache (redis, DB 1=Django cache, DB 2=Celery broker/backend, DB 3=Anubis)
        ├── celery_worker  (background jobs)
        ├── celery_beat    (job scheduler)
        └── powersync      (mobile offline-sync service, port 8080)
```

Services are split across included files: `services/postgres.yaml`, `services/powersync.yaml`, `services/redis.yaml`. The root `docker-compose.yml` overrides/extends what those files define.

## Configuration

All environment variables live in `config/`. The layering order for dev environments:

1. `config/prod.env` — base config, most settings live here
2. `config/dev.env` — overrides: debug mode, disables axes/celery, sets dev keys
3. `config/dev-sqlite.env` — additional overrides for SQLite path and concurrency

For production customisation, copy `docker-compose.override.example.yml` to `docker-compose.override.yml` and add a `config/wger-local.env` with only the changed variables.

## Key Configuration Notes

- **`SECRET_KEY`** and **`JWT_PUBLIC_KEY`/`JWT_PRIVATE_KEY`**: The defaults in `prod.env` must be changed for production. Generate JWT keys with `./manage.py generate-jwt-keys`.
- **PowerSync**: Requires `wal_level=logical` on Postgres (already set in `services/postgres.yaml`). The JWKS URL (`PS_JWKS_URL`) must point to the web service.
- **`WGER_CODEPATH`**: Required by both dev compose files — absolute path to the wger Django source checkout on the host.
- **Port overrides**: Dev environments expose `WGER_HOST_PORT` (default 8000), `POSTGRES_HOST_PORT` (default 5432), and `POWERSYNC_HOST_PORT` (default 8080) via `.env` files.

## Optional Overlays

- **Anubis** (bot protection): Example config in `docker-compose.override.example.yml`. Sits in front of the web service on port 3000; rules in `config/anubis-rules.yml`. The API paths (`/api/v2`, `/allauth`, `/identity/`) are always ALLOWed.
- **Caddy**: Commented-out example in the override file; replaces nginx but nginx must remain (serves static files).
- **Celery Flower**: Monitoring UI on port 5555, also in the override example.

## Dokploy Deployment

Two files handle the Dokploy/Traefik integration without touching the base `docker-compose.yml`:

- **`docker-compose.dokploy.yml`** — compose override that removes nginx's host port binding, attaches it to `dokploy-network`, and adds Traefik labels.
- **`config/dokploy.env`** — env overrides that enable `X_FORWARDED_PROTO_HEADER_SET` and `USE_X_FORWARDED_HOST` so Django trusts Traefik's HTTPS headers (prevents CSRF errors).

**Setup in Dokploy UI:**
1. Compose file: `docker-compose.yml`
2. Compose override: `docker-compose.dokploy.yml`
3. Environment variable: `WGER_DOMAIN=your-domain.com`
4. Edit `config/dokploy.env`: set `SITE_URL=https://your-domain.com`

nginx stays in the stack to serve `static` and `media` volumes — only it joins `dokploy-network`; all other services remain on the internal `wger_network`.

## Tests

The `tests/` directory contains a pytest suite that validates nginx proxy header behavior across reverse-proxy, direct, port-forward, and WebSocket scenarios. It uses a mock Python HTTP backend (`mock_backend.py`) instead of the real wger app. CI runs these tests on changes to `config/nginx.conf` or `tests/**`.
