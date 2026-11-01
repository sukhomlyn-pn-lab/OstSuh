# Project: SuccessfulSuccess Meetings (Monorepo)

## Purpose and Scope
This repository contains the local development stack and AWS deployment configuration for a per-user meetings application. The browser client lists, creates, edits, and deletes meetings for the signed-in user. Authentication is provided by Amazon Cognito.

This document describes the implementation currently in the repository. It differs from the initial lab example: the frontend is Next.js rather than Vite, meeting attendees are represented as participant records rather than an `attendee_count` input, and the AWS deployment templates use Lambda and Aurora rather than ECS and an Application Load Balancer.

## Repository Structure
- `backend/` — Python API service. `app/api/` contains HTTP routes and dependencies; `app/schemas/` defines request/response validation; `app/models/` defines SQLAlchemy entities; `app/repositories/` handles database queries; `app/services/` contains meeting and user operations; `alembic/` contains versioned database migrations; `tests/` contains backend tests. `Dockerfile` is the local Compose image; `Dockerfile.lambda` packages the API for AWS Lambda.
- `frontend/` — Next.js App Router application. `app/` contains routes and global styles; `components/` contains page, meeting, authentication, and UI components; `hooks/` contains client hooks; `lib/` contains API, auth, date-time, and shared type utilities; `public/` contains static assets. Its Dockerfile has development, production, and static-export targets.
- `infra/` — AWS CloudFormation templates for Cognito authentication, ECR, the backend, and the frontend.
- `.github/workflows/` — GitHub Actions workflow for backend Ruff and frontend ESLint/style checks on pushes to `main`.
- `docker-compose.yml` — Local orchestration for PostgreSQL, backend, and frontend.
- `Makefile` — Local development, test, lint, migration, seed, and AWS deployment commands.
- `README.md` — Setup, local operation, authentication, and deployment instructions.
- `SPEC.md` — Product requirements and behavioral specification.

## Runtime and Docker Contracts
Compose builds the backend and frontend from their service Dockerfiles and runs PostgreSQL from the official image.

| Service | Image/build base | Container port | Host port | Readiness and dependency |
| --- | --- | ---: | ---: | --- |
| `db` | `postgres:16-alpine` | 5432 | `${POSTGRES_PORT:-5432}` | `pg_isready` runs every 5 seconds, with a 5-second timeout and 10 retries. Database data persists in the `pgdata` named volume. |
| `backend` | `python:3.12-slim`; copies `uv` from `ghcr.io/astral-sh/uv:latest` | 8000 | `${BACKEND_PORT:-8000}` | Waits for `db` with `condition: service_healthy`; applies Alembic migrations at startup when `RUN_MIGRATIONS_ON_START=true`; healthcheck calls `/health` every 10 seconds, with a 20-second start period. |
| `frontend` | `node:22-alpine` | 3000 | `${FRONTEND_PORT:-3000}` | Depends on backend with `condition: service_started`; this orders startup but does not wait for backend health. |

The backend connects to PostgreSQL over the Compose network at `db:5432`; its database URL is assembled from `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB`. The host port can be changed independently, for example to 5433 when host port 5432 is already in use. The browser uses `NEXT_PUBLIC_API_BASE_URL`, which defaults to `http://localhost:8000`.

The application image base tags are specified above. The copied `uv` image currently uses the floating `latest` tag, so not every image reference is version-pinned.

## API Contract
The API is rooted at `/api/v1`. Meeting endpoints require `Authorization: Bearer <Cognito access token>` and operate only on the authenticated user's meetings. Timestamps are ISO-8601 strings and must include a UTC offset. Response timestamps are serialized in the configured application timezone.

### `GET /api/v1/meetings`
Lists meetings overlapping a calendar day. Query parameters:
- `date` — optional `YYYY-MM-DD`; defaults to today in `APP_TIMEZONE`.
- `q` — optional search text, up to 200 characters.
- `limit` — optional page size, default 100, range 1–500.
- `offset` — optional zero-based offset, default 0.

Returns HTTP 200 with a paginated object:

```json
{
  "items": [
    {
      "id": "64c10442-643f-42de-9ab1-2db1c5f8c7bd",
      "name": "Design Sync",
      "description": "Review the next release",
      "location": "Room 3",
      "starts_at": "2026-10-27T10:00:00+03:00",
      "ends_at": "2026-10-27T11:00:00+03:00",
      "participants": [
        {
          "id": "e69ff6d9-2237-4ee9-87a3-e7c3d22067d4",
          "name": "Alex Example",
          "email": "alex@example.com",
          "position": 0
        }
      ],
      "created_at": "2026-10-26T14:00:00+03:00",
      "updated_at": "2026-10-26T14:00:00+03:00"
    }
  ],
  "total": 1,
  "limit": 100,
  "offset": 0,
  "date": "2026-10-27"
}
```

### `POST /api/v1/meetings`
Creates a meeting for the authenticated user. Request body:

```json
{
  "name": "Design Sync",
  "description": "Review the next release",
  "location": "Room 3",
  "starts_at": "2026-10-27T10:00:00+03:00",
  "ends_at": "2026-10-27T11:00:00+03:00",
  "participants": [
    { "name": "Alex Example", "email": "alex@example.com" }
  ]
}
```

`description` and `location` are optional and nullable. `participants` defaults to an empty list and accepts at most 50 unique participants; each participant has a required nonblank `name` (maximum 120 characters) and optional valid `email`. `name` is required and limited to 200 characters. `ends_at` must be later than `starts_at`.

Returns HTTP 201 with the created meeting in the `MeetingRead` response shape shown above and a `Location: /api/v1/meetings/{id}` header. Invalid input returns HTTP 422. Requests without valid authentication are rejected; callers cannot access another user's meetings.

`GET /health` is an unauthenticated service health endpoint. It returns HTTP 200 when the API can query the database and HTTP 503 when the database is unreachable.

## Startup Flow
Compose starts PostgreSQL and waits for its `pg_isready` healthcheck before starting the backend. The backend runs Alembic migrations during container startup, then serves HTTP on port 8000. Compose starts the frontend after the backend container starts; the frontend does not wait for the backend healthcheck. The developer runs the stack with `docker compose up --build` after installing Docker Desktop.
