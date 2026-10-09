# CampusForge API

CampusForge is a concurrency-safe campus workshop registration REST API built for the AWS Student Builder Group Web Development challenge.

The project goes beyond basic CRUD. It models **Users → Workshops → Registrations**, enforces role-based access, uses short-lived access tokens with rotating refresh tokens, and protects the workshop capacity invariant with a PostgreSQL transaction and row-level locking.

## What the project demonstrates

- **Relational data model:** `users`, `workshops`, and `registrations` with foreign keys and database constraints.
- **RBAC:** `ADMIN`, `ORGANIZER`, and `STUDENT` roles with endpoint-level authorization.
- **Authentication:** short-lived JWT access tokens plus rotating, revocable refresh tokens.
- **Concurrency safety:** registration is serialized per workshop with PostgreSQL `SELECT ... FOR UPDATE`.
- **Duplicate protection:** `UNIQUE(workshop_id, user_id)` prevents the same student from registering twice.
- **Idempotency:** clients can provide an `Idempotency-Key` when retrying registration requests.
- **List endpoint features:** pagination, search filtering, and allow-listed sorting.
- **Automated tests:** includes an actual concurrent-registration test, not just status-code smoke tests.

## Stack

Node.js 20+, Express 4, PostgreSQL 16+ (the project was tested locally with PostgreSQL 17), JWT, bcryptjs, Zod, Jest, and Supertest.

## Quick start

### Prerequisites

You need:

- Node.js 20 or newer
- PostgreSQL 16 or newer

Docker Desktop is **optional**. It is included as a convenient reproducible database option, but it is not required when PostgreSQL is already installed locally.

### Option A — use a local PostgreSQL installation

Create a database named `campusforge`, then create `.env` from the example:

```cmd
copy .env.example .env
```

Edit `.env` and set the PostgreSQL password in `DATABASE_URL`, plus strong JWT secrets:

```env
PORT=4000
DATABASE_URL=postgres://postgres:YOUR_PASSWORD@localhost:5432/campusforge
JWT_ACCESS_SECRET=replace-with-a-long-random-access-secret
JWT_REFRESH_SECRET=replace-with-a-long-random-refresh-secret
ACCESS_TOKEN_TTL=15m
REFRESH_TOKEN_TTL_DAYS=7
NODE_ENV=development
```

Then install dependencies, create the schema, seed demo data, and start the API:

```bash
npm install
npm run migrate
npm run seed
npm run dev
```

Health check:

```text
GET http://localhost:4000/health
```

### Option B — use Docker Desktop for PostgreSQL

Start only the database container:

```bash
docker compose up -d
```

The bundled Compose file creates a local `campusforge` database on port `5432` with the demo password `postgres`.

Then set:

```env
DATABASE_URL=postgres://postgres:postgres@localhost:5432/campusforge
```

and run:

```bash
npm install
npm run migrate
npm run seed
npm run dev
```

## Demo accounts

All seeded demo accounts use the password `Builder@123`.

| Role | Email |
|---|---|
| ADMIN | `admin@campusforge.local` |
| ORGANIZER | `organizer@campusforge.local` |
| STUDENT | `student@campusforge.local` |

Public registration always creates a `STUDENT`; privileged roles are not self-assignable.

## API surface

### Authentication

- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`

### Workshops

- `GET /api/workshops?page=1&limit=10&search=aws&sort=starts_at&order=asc`
- `GET /api/workshops/:id`
- `POST /api/workshops` — `ADMIN` / `ORGANIZER`
- `PATCH /api/workshops/:id` — `ADMIN` / owner `ORGANIZER`
- `DELETE /api/workshops/:id` — `ADMIN` / owner `ORGANIZER`

### Registrations

- `POST /api/workshops/:id/register` — authenticated user
- `DELETE /api/workshops/:id/register` — authenticated user
- `GET /api/workshops/:id/registrations` — `ADMIN` / `ORGANIZER`

A registration request may include:

```http
Idempotency-Key: a-client-generated-key-123
```

See `API_EXAMPLES.md` for request examples.

## Concurrency design

The most important write path is workshop registration. A simple application-level check like:

```text
if seats_held < capacity:
    insert registration
```

is unsafe because two concurrent requests can both read the same available seat before either writes.

CampusForge instead performs the critical section inside one PostgreSQL transaction:

```text
BEGIN
  |
  +-- SELECT workshop ... FOR UPDATE
  |
  +-- check seats_held < capacity
  |
  +-- INSERT registration
  |
  +-- increment seats_held
  |
COMMIT
```

`FOR UPDATE` locks the workshop row until the transaction completes. A second registration attempt for the same workshop waits, then re-reads the current seat count. This makes the database—not an application-level check—the authority for the capacity invariant.

The automated concurrency test creates a workshop with capacity `3` and fires `8` registration requests concurrently. Exactly `3` should succeed and the remaining `5` should receive `409 WORKSHOP_FULL`.

## Tests

Run:

```bash
npm test
```

The suite covers:

1. Health endpoint behavior.
2. RBAC enforcement.
3. Workshop creation plus pagination/filtering.
4. Duplicate-registration protection.
5. Concurrent registration never exceeding workshop capacity.

The tests use a real PostgreSQL database and clean up their own test records.

## Suggested reviewer demo

1. Log in as the organizer.
2. Create a workshop with a small capacity such as `3`.
3. Register students until capacity is reached.
4. Show that another registration returns `409 WORKSHOP_FULL`.
5. Run `npm test` and point out the concurrent-registration test.
6. Explain why `SELECT ... FOR UPDATE` is needed instead of an application-level `if` check.
7. Show the three roles and explain which endpoints each role can access.

## Project structure

```text
src/
  config/       environment + PostgreSQL pool
  controllers/  HTTP request/response layer
  middleware/   authentication, RBAC, request IDs, errors
  routes/       API route definitions
  services/     business logic and transactions
  utils/        JWT and HTTP helpers

db/
  schema.sql    relational schema + constraints
  migrate.js    schema setup
  seed.js       demo data

tests/
  api.test.js   integration tests
```

## Security notes

This is a challenge/demo project. For production deployment, use a managed secrets store, HTTPS-only secure refresh-token transport, stricter CORS, rate limiting, audit logging, and a production-grade migration/versioning system.

**Never commit `.env` or other secrets.** The repository `.gitignore` excludes `.env`, `node_modules/`, coverage output, and log files.
