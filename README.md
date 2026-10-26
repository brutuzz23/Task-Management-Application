# WorkSync API

Task-management backend: **Java 21, Spring Boot 3.3, PostgreSQL, JWT, Flyway, Docker**.

## Run locally

```bash
cp .env.example .env          # set JWT_SECRET (32+ chars)
docker compose up --build     # API :8080, Postgres :5432, MailHog UI :8025
```

Or run the app from your IDE / `mvn spring-boot:run` with only `db` and `mailhog` up:
`docker compose up db mailhog`.

## API

| Method | Path | Notes |
|---|---|---|
| POST | `/api/auth/register` | `{email, password}` → `{token, userId, email}` |
| POST | `/api/auth/login` | same shape |
| POST | `/api/auth/forgot-password` | `{email}` → 202 always; emails a single-use link (30 min) |
| POST | `/api/auth/reset-password` | `{token, newPassword}` → 204 |
| GET/POST | `/api/boards` | list / create your boards |
| GET/PUT/DELETE | `/api/boards/{boardId}` | owner only (others get 404) |
| GET/POST | `/api/boards/{boardId}/tasks` | `?status=TODO\|IN_PROGRESS\|DONE` |
| GET/PUT/DELETE | `/api/boards/{boardId}/tasks/{taskId}` | owner of the board only |
| GET | `/actuator/health` | |

Send `Authorization: Bearer <token>` on everything except `/api/auth/**`.

```bash
TOKEN=$(curl -s localhost:8080/api/auth/register -H 'Content-Type: application/json' \
  -d '{"email":"me@example.com","password":"password123"}' | jq -r .token)
curl -s localhost:8080/api/boards -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d '{"name":"Sprint 1"}'
```

## Design notes

- **Access control**: every board/task query is scoped to the authenticated user's id; non-owners get `404`, not `403`.
- **Soft deletion**: `@SQLDelete` + `@SQLRestriction` on `Board`/`Task` set `deleted_at`; deleting a board cascades to its tasks.
- **Password recovery**: random 256-bit token, only its SHA-256 is stored, single use, 30-minute expiry. Reset emails land in MailHog locally.
- **Migrations**: Flyway in `src/main/resources/db/migration`; Hibernate runs with `ddl-auto: validate`.

## Tests

`mvn verify` — unit tests plus an integration suite on real PostgreSQL via Testcontainers (needs Docker).

## CI/CD (`.github/workflows`)

1. `ci.yml` — build + tests on every push/PR.
2. `static-analysis.yml` — CodeQL security-and-quality scan (push, PR, weekly).
3. `deploy.yml` — builds and pushes the image to GHCR; the SSH deploy job runs only if repo variable `DEPLOY_ENABLED=true` and secrets `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY` exist (server needs `/opt/worksync/docker-compose.yml` + `.env` with `APP_IMAGE`).
