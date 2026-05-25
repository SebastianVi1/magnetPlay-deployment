# MagnetPlay — Agent Guide

## Repo structure

Root is a **docker-compose orchestration layer** over 4 git submodules.

| Service | Dir | Stack | Port |
|---|---|---|---|
| Frontend | `MagnetPlay-Frontend/` | React 19 + Vite + TypeScript | `:5173` (:80 prod) |
| Backend | `MagnetPlay-Backend/` | Spring Boot 3.5 + Java 21 + Maven + PostgreSQL | `:8080` |
| Torrent API | `Torrent-Api-py/` | FastAPI + Python 3.8‑3.11 | `:8009` |
| Streaming API | `StreamTorrent-Api/` | Express + WebTorrent + Node 18 | `:3000` |

## First clone

```bash
git clone --recurse-submodules ...
# or after cloning:
git submodule update --init --recursive
```

## Updating

```bash
git pull origin master
git submodule update --remote --merge --recursive
# then rebuild/recreate containers
```

## Development

```bash
docker compose up --build -d
```

Frontend at `http://localhost:5173`, backend at `http://localhost:8080`.

Vite dev container uses live-reload (`usePolling: true`, `src/` volume mount). Backend dev container runs via `mvn spring-boot:run` with devtools (port `35729`). Streaming API runs via `nodemon`.

## Production

```bash
export TAG='v1.0.0'
docker compose -f docker-compose.prod.yml --env-file ./.env.prod build --no-cache
docker compose -f docker-compose.prod.yml --env-file ./.env.prod up -d
```

Frontend served via `serve` on port 80. Backend built as a fat JAR (`mvn package -DskipTests`), run with `SPRING_PROFILES_ACTIVE=production`. Streaming API runs `node src/server.js`.

## Environment

- `.env.dev` — dev compose vars (Postgres creds, service URLs)
- `.env.prod` — production compose vars (**not committed**, in `.gitignore`)
- `.env.prod.example` — template for prod env
- `MagnetPlay-Frontend/.env.development` / `.env.production` — Vite‑side build‑time vars

`.env.prod` contains secrets; never commit it.

## CORS

Backend allows a single CORS origin via `app.frontend.url` (set by `FRONTEND_URL` env). In dev, this must be `http://localhost:5173` — the browser-facing Vite URL, **not** the Docker hostname. In prod, it's the public domain.

## Submodule caveats

- Submodules track their upstream `main` branch. Changes to submodule files in this repo will conflict unless committed in the submodule first.
- `Torrent-Api-py/` has its own `.git` (real submodule); `StreamTorrent-Api/` and `MagnetPlay-Backend/` also do. Do not edit files inside submodules unless you intend to push changes upstream.

## Key endpoints

- **Torrent API**: `http://torrent-api:8009` — search, trending, recent, category endpoints under `/api/v1/`. Auth via `x-api-key` header (`PYTORRENTS_API_KEY` env).
- **Streaming API**: `GET /api/torrent/:magnet` — streams video from magnet link. **Requires `Range` header.** Chunked responses (default 4 MB). Max 20 active torrents, 10 min inactivity timeout.
- **Backend**: Spring Data JPA REST API with JWT auth. Swagger at `/swagger-ui.html`. OAuth2 + JWT flow.
 - **Frontend**: Vite proxies `/api` to backend (`:8080`) and `/api/torrent` to streaming API (`:3000`). **Proxy rule order matters** — `/api/torrent` must be listed before `/api` in `vite.config.ts` or `/api` catches it first.

## Torrent scrapers

**Primary site: Glodls** (`glodls`). Other working scrapers: `torlock`, `nyaasi`, `torrentproject`. PirateBay and 1337x are Cloudflare-blocked. Glodls supports search, trending, and recent without a `category` param.

**Cloudscraper bypass**: `Torrent-Api-py/torrents/x1337.py` uses `cloudscraper` (async via `run_in_executor`) for Cloudflare-challenged pages. Python 3.11+ required (dev Dockerfile updated from 3.8).

**Backend integration**: `MovieService.java` reads `app.torrentapi.url` from properties (was previously hardcoded). `RestTemplate` is injected via `RestTemplateConfig` with 5s connect / 30s read timeouts. Backend uses `piratebay` as the primary site for recent/trending. Search uses `piratebay` by default.

**Backend error handling**: `GlobalExceptionsHandler` now returns HTTP 503 with `"Torrent API Error"` when `RestClientException` is thrown (was previously returning misleading 404 "The user entered was not found").

## Testing

- **Backend**: `mvn test` (inside `MagnetPlay-Backend/`)
- **No tests** for Frontend, Torrent API, or Streaming API

## Backend build (local, outside Docker)

```bash
cd MagnetPlay-Backend
./mvnw clean install   # or mvn if installed
./mvnw spring-boot:run
```

## Networking

All services communicate via docker compose internal network (e.g. `http://backend:8080`, `http://streaming-api:3000`). The Vite dev server proxies `/api` and `/api/torrent` so the frontend dev container hits both through `localhost:5173`.

## Docker images

Published to Docker Hub under `sebastianvi1/`:
- `magnetplay-frontend`, `magnetplay-backend`, `torrent-scrapper`, `streaming-api`
- Dev images tagged `:latest-dev`; prod images tagged with `$TAG` (default `:latest`)
