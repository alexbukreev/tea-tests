# Tea Tasting Platform — Technical Plan (MVP, 3 sprints)

## 0. Overview

- Domain: collective tea tastings.
- Frontend: React + Vite + shadcn/ui.
- Backend: Node.js (Express or lightweight NestJS).
- Database: PostgreSQL.
- Auth: Telegram-based, tokenized links for web access.
- Infra:
  - Dev: local Postgres in Docker, frontend/backend run natively.
  - Prod: single Docker app (API + static frontend) on hosting platform + managed PostgreSQL instance.

Work is split into three 2-week sprints. This document describes the target architecture and technical scope per sprint.

---

## 1. Architecture

### 1.1. Components

- **Frontend app**
  - React + Vite.
  - UI kit: shadcn/ui.
  - Consumes REST API.
  - Main flows:
    - rating form with radar chart;
    - participant result view;
    - minimal admin UI.

- **Backend API**
  - Node.js application (Express or NestJS).
  - Responsibilities:
    - Telegram webhook/handler endpoints;
    - user registration & identification via Telegram;
    - auth link generation & resolution;
    - CRUD for tastings, tea samples, ratings;
    - aggregation for reports.

- **Database**
  - PostgreSQL 16/18.
  - Managed instance in production.
  - Local Docker container in development.

- **Telegram bot**
  - Simple bot using `node-telegram-bot-api` or Telegraf (or any other preferred library).
  - Talks only to backend API; backend owns all business logic.

- **Deployment**
  - Dockerfile for backend (serves static frontend build).
  - docker-compose:
    - `docker-compose.dev.yml` — local dev (db only).
    - `docker-compose.yml` — production (app only; DB is external managed instance).
  - CI/CD or platform autodeploy from Git repo.

---

## 2. Data model (MVP)

### 2.1. Tables

Minimal schema (conceptual):

- **users**
  - `id` (UUID / serial PK)
  - `telegram_id` (bigint, unique)
  - `telegram_username` (text, nullable)
  - `telegram_full_name` (text)
  - `is_admin` (boolean, default false)
  - `created_at` (timestamp)

- **tastings**
  - `id` (UUID PK)
  - `title` (text)
  - `description` (text, nullable)
  - `scheduled_at` (timestamp, nullable)
  - `created_at` (timestamp)
  - `created_by` (FK → users.id, nullable)

- **tea_samples**
  - `id` (UUID PK)
  - `tasting_id` (FK → tastings.id)
  - `name` (text)
  - `notes` (text, nullable)
  - `order_index` (int) — order in tasting.

- **rating_dimensions**
  - `id` (UUID PK)
  - `tasting_id` (FK → tastings.id)
  - `code` (text) — e.g. `aroma`, `sweetness`
  - `label` (text) — human-readable label
  - `min_value` (int, default 0)
  - `max_value` (int, default 10)

- **ratings**
  - `id` (UUID PK)
  - `user_id` (FK → users.id)
  - `tea_sample_id` (FK → tea_samples.id)
  - `created_at` (timestamp)
  - `data` (jsonb) — map: dimension_code → value (int)

- **auth_links**
  - `id` (UUID or random token PK)
  - `user_id` (FK → users.id)
  - `purpose` (enum: `rating_page`, `result_page`, `admin_panel`, ...)
  - `context` (jsonb) — e.g. `{ "tasting_id": "...", "tea_sample_id": "..." }`
  - `expires_at` (timestamp)
  - `used_at` (timestamp, nullable)

This schema is BI-friendly (especially `ratings.data` with dimension codes) and flexible enough for future changes.

---

## 3. Auth & flows

### 3.1. Telegram user registration

- Bot receives `/start` or any command.
- Bot sends POST to backend:

  `POST /api/telegram/register`

  ```json
  {
    "telegram_id": 123456789,
    "username": "user",
    "full_name": "Tea Lover"
  }
  ```

- Backend logic:
  - `users.find_or_create_by(telegram_id)`.
  - Save/update username/full_name.
  - Return `{ "status": "ok" }`.

### 3.2. Auth links (one-time / short-lived)

- Backend endpoint:

  `POST /api/auth/link`

  Body example (for rating page):

  ```json
  {
    "telegram_id": 123456789,
    "purpose": "rating_page",
    "context": {
      "tasting_id": "UUID",
      "tea_sample_id": "UUID"
    }
  }
  ```

- Backend:
  - resolve user by `telegram_id`;
  - create `auth_links` entry with `expires_at` (e.g. +30 minutes);
  - return URL: `https://app.example/rate?token=<auth_link_id>`.

- Bot sends this URL to user.

### 3.3. Auth link resolution

- Frontend:
  - reads `token` from query;
  - calls `GET /api/auth/resolve?token=<token>`.

- Backend:
  - validate token:
    - exists,
    - not expired,
    - `purpose` matches expected (optional check),
  - optionally set `used_at` (for strict one-time links),
  - return:

  ```json
  {
    "user": {
      "id": "UUID",
      "name": "Tea Lover"
    },
    "purpose": "rating_page",
    "context": {
      "tasting_id": "UUID",
      "tea_sample_id": "UUID"
    }
  }
  ```

- Frontend:
  - stores minimal session info (userId, tastingId, teaSampleId) in memory / session storage;
  - uses them for subsequent API calls.

### 3.4. Admin access

- `users.is_admin = true` marks admins.
- Admin flow:
  - admin sends `/admin` to bot;
  - bot requests auth link with `purpose = "admin_panel"`;
  - backend verifies user is admin, creates link;
  - admin opens `https://app.example/admin?token=...`;
  - frontend resolves token via `/api/auth/resolve` and checks `purpose === "admin_panel"`.

---

## 4. Sprint breakdown (technical)

### Sprint 1 — Core backend, DB, Telegram registration, initial deploy

**Goals:**

- Set up repo structure, base backend, DB schema, Telegram integration, and initial deploy with Docker.

**Tasks:**

1. **Repo & tooling**
   - Create monorepo or two repos (`frontend`, `backend`) — to be decided.
   - Setup TypeScript for backend (optional but recommended).
   - Setup basic linting/formatting.

2. **Backend skeleton**
   - Express or NestJS app.
   - Basic routes:
     - `GET /health` — health check.
   - Config loader (env-based) for DB URL, Telegram token, etc.

3. **Database**
   - Choose ORM/migrations tool (Prisma / Knex / TypeORM).
   - Implement initial migration:
     - `users`,
     - `tastings`,
     - `tea_samples`.

4. **Telegram integration**
   - Create bot and get token.
   - Implement `/api/telegram/register` endpoint.
   - Implement simple bot flow:
     - `/start` → call `/api/telegram/register`.

5. **Dev infra**
   - `docker-compose.dev.yml`:
     - `db` (Postgres).
   - Backend connects to `postgres://...@localhost:5432/tea_dev`.

6. **Prod infra (first iteration)**
   - Dockerfile for backend (build + start).
   - `docker-compose.yml` for production app service (single `app` container).
   - Connect app to managed Postgres instance via env `DATABASE_URL`.
   - Deploy to hosting platform, ensure `/health` works.

**Deliverables:**

- Working backend with DB and Telegram registration.
- Hosted app (API only + placeholder frontend page).
- Technical documentation for env variables, DB connection, and Telegram setup.

---

### Sprint 2 — Tastings, ratings, rating UI with radar charts

**Goals:**

- Implement full rating flow: tastings, samples, dimensions, ratings.
- Implement auth links and web rating page with radar chart.

**Tasks:**

1. **DB extensions**
   - Add tables:
     - `rating_dimensions`,
     - `ratings`,
     - `auth_links`.
   - Migrations.

2. **Backend API**
   - CRUD endpoints (can be admin-only for now):
     - `POST /api/tastings`
     - `POST /api/tastings/:id/samples`
     - `POST /api/tastings/:id/dimensions`
   - Auth link endpoints:
     - `POST /api/auth/link`
     - `GET /api/auth/resolve`
   - Rating endpoints:
     - `POST /api/ratings` — create/update rating for user/sample.

3. **Telegram → rating link**
   - Bot flow to:
     - list or pick tasting,
     - request rating link for a given tasting/sample,
     - send link to user.

4. **Frontend rating page**
   - Vite + shadcn/ui setup.
   - Implement rating page:
     - reads `token` from URL;
     - calls `/api/auth/resolve`;
     - fetches tasting context (sample name, dimensions);
     - shows input controls (sliders/inputs) for each dimension;
     - shows radar chart (Recharts or Chart.js) bound to current values;
     - sends ratings via `POST /api/ratings`.

5. **UX & testing**
   - Test full flow end-to-end with test tasting and few users.
   - Fix obvious usability issues.

**Deliverables:**

- Complete rating flow from Telegram to web and back to backend.
- Users can rate tea samples; data is persisted.
- Radar chart UI working on desktop and mobile.

---

### Sprint 3 — Participant reports, admin panel, analytics preparation

**Goals:**

- Provide participant reports (profile + group comparison).
- Implement minimal admin UI for managing tastings and exporting data.
- Prepare DB for BI tools (DataLens).

**Tasks:**

1. **Backend analytics endpoints**
   - Aggregations:
     - `GET /api/tastings/:id/summary`
       - average values per dimension, per sample;
     - `GET /api/users/:id/tastings/:tastingId/profile`
       - user ratings + group averages.
   - Business logic:
     - compute basic textual profile for user (simple rules based on dimension preferences).

2. **Participant report page**
   - Frontend route for result page.
   - Auth via token (`purpose = "result_page"`).
   - UI:
     - radar chart: user vs group;
     - textual profile/resume.

3. **Bot integration for results**
   - After tasting completion (manual trigger at MVP), admin can initiate sending result links.
   - Backend generates auth links for `result_page`.
   - Bot sends links to participants.

4. **Admin panel (minimal)**
   - Admin route `/admin`.
   - Auth via token (`purpose = "admin_panel"`, user.is_admin = true).
   - Features:
     - list tastings,
     - create/edit tasting, samples, dimensions,
     - view participation status,
     - export ratings as CSV.

5. **DataLens preparation**
   - Document DB tables and fields for BI:
     - which tables to connect,
     - how to explode `ratings.data` for dimensions (if needed).
   - Optional: simple SQL views to simplify BI usage.

6. **Infra polish**
   - Separate env configs for dev/prod.
   - Basic logging and error handling.
   - Update deploy instructions.

**Deliverables:**

- Participant report pages with comparison vs group and basic profile texts.
- Admin panel with ability to manage tastings and export data.
- Documentation on how to connect DB to BI tools (DataLens).

---

## 5. Environments & configuration

### Development

- Local machine (macOS).
- Tools:
  - Node.js (LTS),
  - Docker Desktop,
  - VS Code / any editor.
- Services:
  - `docker-compose.dev.yml`:
    - `db` (Postgres, port 5432).
- Backend:
  - `.env.local` with `DATABASE_URL`, `TELEGRAM_BOT_TOKEN`, etc.
- Frontend:
  - Vite dev server (port 5173 by default).

### Production

- Managed PostgreSQL instance (host, port, DB name, user, password).
- App hosting service that runs Docker Compose / Docker app:
  - Image built from Dockerfile in repo.
- Env variables:
  - `DATABASE_URL`
  - `TELEGRAM_BOT_TOKEN`
  - `APP_BASE_URL`
  - etc.
- Autodeploy from Git repo (on push to main/master).

---

## 6. Next steps after MVP

- Enhance rating model (text reviews, more dimensions).
- Advanced recommendation engine for teas based on profiles.
- Real-time dashboards in DataLens.
- Optional web3/crypto extensions (on-chain proofs of tastings, etc.) if needed in future.
