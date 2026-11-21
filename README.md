# FeedMe

FeedMe is a full-stack social feed demo built to explore how a modern feed system balances freshness, personalization, caching, async propagation, and observability in a small but production-style app.

The repository contains:

- `frontend/`: React 19 + TypeScript dashboard for browsing feeds and user actions
- `backend/`: Spring Boot 4 API that serves feed, timeline, follow, and post operations
- `docker-compose.yml`: full local stack with PostgreSQL, Redis, Prometheus, Grafana, Loki, and containerized app services

## Stack

- Frontend: React 19, TypeScript, Vite
- Backend: Spring Boot 4.1, Java 17
- Data: PostgreSQL, Redis
- Metrics and logs: Micrometer, Prometheus, Grafana, Loki, Promtail
- App runtime: Docker Compose for local orchestration

## What the app does

- create posts with idempotency protection
- fetch a personalized home feed
- fetch a user timeline
- follow and unfollow users
- browse user lists and suggested accounts
- paginate feed results with cursor-based keyset pagination
- cache the first home-feed page in Redis
- publish feed events through an outbox + Redis Streams pipeline
- support a hybrid ranking strategy for hot users
- expose Prometheus-compatible metrics via Actuator
- seed demo users, follows, and posts on startup

## Repository layout

```text
FeedMe/
├── backend/
│   ├── Dockerfile
│   ├── mvnw / mvnw.cmd
│   ├── pom.xml
│   └── src/
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── nginx.conf
│   └── src/
├── grafana/
├── loki/
├── prometheus/
├── promtail/
├── docker-compose.yml
├── LICENSE
├── README.md
└── .gitignore
```

## Quick start with Docker Compose

This is the easiest way to run the full stack.

```bash
docker compose up --build
```

Then open:

- Frontend: http://localhost:5173
- Backend API: http://localhost:8080
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (`admin` / `admin`)
- Loki: http://localhost:3100

To stop everything:

```bash
docker compose down
```

To remove volumes for a clean reset:

```bash
docker compose down -v
```

## Running locally without Docker

Start the database and cache first:

```bash
docker compose up postgres redis -d
```

### Backend

```bash
cd backend
./mvnw spring-boot:run
```

Default local config is in `backend/src/main/resources/application.yaml`.

Default values:

- PostgreSQL: `jdbc:postgresql://localhost:5432/feedme`
- DB username: `feedme`
- DB password: `feedme`
- Redis host: `localhost`
- Redis port: `6379`

### Frontend

```bash
cd frontend
npm install
npm run dev
```

The Vite dev server runs on `http://localhost:5173` and proxies `/api` requests to the backend at `http://localhost:8080`.

## Core API

```http
GET    /api/users
GET    /api/follows?followerId=...
POST   /api/follows/{userId}?followerId=...
DELETE /api/follows/{userId}?followerId=...
POST   /api/posts
GET    /api/posts/{postId}
GET    /api/feed/home?userId=...&cursor=...&limit=...
GET    /api/feed/user/{userId}?cursor=...&limit=...
GET    /actuator/health
GET    /actuator/metrics
GET    /actuator/prometheus
```

## Example requests

Get the home feed for viewer `u1`:

```http
GET http://localhost:8080/api/feed/home?userId=u1&limit=5
```

Get a user's timeline:

```http
GET http://localhost:8080/api/feed/user/u2?limit=5
```

Create a post:

```http
POST http://localhost:8080/api/posts
Content-Type: application/json

{
  "idempotencyKey": "create-post-u1-001",
  "authorId": "u1",
  "content": "The personalized feed stays stable while more posts arrive."
}
```

Follow a user:

```http
POST http://localhost:8080/api/follows/u2?followerId=u1
```

## Demo users and seeded data

On startup, the backend seeds a small social graph for demo purposes. The default users are:

- `u1` — Ava Chen
- `u2` — Marcus Vale
- `u3` — Zoya Singh
- `u4` — Lina Tina

The frontend lets you switch the active viewer via the query param `?user=<id>`, so you can quickly compare home-feed behavior across accounts.

## Observability and monitoring

The repo includes a monitoring stack for the feed service:

- Prometheus scrapes app and exporter targets on `:9090`
- Grafana is provisioned for visual dashboards on `:3000`
- Loki collects container logs via Promtail
- Spring Boot actuator exposes health, metrics, and Prometheus endpoints

The backend metrics include feed-related timing and cache behavior, and the app surfaces them from:

```http
GET http://localhost:8080/actuator/prometheus
```

## Notes

- The frontend is intentionally designed as a single-page dashboard with a live feed view, composer, follower lists, and timeline drill-down.
- The backend uses a hybrid feed strategy: non-hot and hot author slices are merged to keep ranking useful without starving the rest of the network.
- The first home-feed page is cached in Redis for a fast warm path, while the system still supports follow and post updates through async event processing.
- The production build for the frontend succeeds with `npm run build`.
