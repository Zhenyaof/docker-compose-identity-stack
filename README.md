# Identity Stack — Postgres + Keycloak + Curl (Docker Compose)

A production-style Docker Compose stack that runs [Keycloak](https://www.keycloak.org/)
backed by PostgreSQL, with a dependent client container gated on both
services being fully healthy — not just "started."

## Architecture

```
┌─────────────┐        ┌─────────────┐
│  postgres   │◄───────│  keycloak   │
│ (healthy)   │  JDBC  │ (healthy)   │
└──────┬──────┘        └──────┬──────┘
       │                      │
       │   both must be       │
       │   service_healthy    │
       └──────────┬───────────┘
                   ▼
            ┌─────────────┐
            │    curl     │
            │ (always up, │
            │ starts last)│
            └─────────────┘
```

- **postgres** — stores Keycloak's realm/user/config data.
- **keycloak** — identity & access management server; only its login
  page (port `9090` → container's `8080`) is exposed to the host.
- **curl** — a lightweight client container that idles forever, but
  only *starts* once both `postgres` and `keycloak` report
  `healthy` via their Docker healthchecks (`depends_on` +
  `condition: service_healthy`).

## Requirements this stack satisfies

| Requirement | How it's implemented |
|---|---|
| Postgres data persisted outside the default volume location | Bind mount `./data/postgres` → `/var/lib/postgresql/data` — lives in the project folder, not Docker's internal `/var/lib/docker/volumes/...` |
| Keycloak stores its data in Postgres | `KC_DB=postgres` + `KC_DB_URL` pointing at the `postgres` service over the internal Docker network |
| Curl always running, but only starts once Postgres **and** Keycloak are healthy | `depends_on: { postgres: {condition: service_healthy}, keycloak: {condition: service_healthy} }`, entrypoint is an infinite idle loop |
| No logs stored as JSON | All three services use the `local` logging driver (binary format), not the default `json-file` driver |
| Logs capped, rotated, never more than 20 files | `max-size` + `max-file: "20"` on every service's logging options |
| Only the Keycloak login page is visible to the user | Only `keycloak` publishes a port to the host (`9090:8080`); Postgres and curl publish nothing |

## Prerequisites

- Docker Desktop (or Docker Engine + Compose plugin)
- Docker Compose v2 (`docker compose`, not the old `docker-compose`)

## Quick start

1. Copy the environment template and fill in your own credentials:

   ```bash
   cp .env.example .env
   ```

   (`.env` is git-ignored — never commit real secrets.)

2. Start the stack:

   ```bash
   docker compose up -d
   ```

3. Watch it come up healthy:

   ```bash
   docker compose ps
   ```

4. Open the Keycloak login page:

   ```
   http://localhost:9090
   ```

   Log in with the `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` you
   set in `.env`.

Full command reference (health checks, log inspection, teardown,
rebuilding) is in [`docker-commands.md`](./docker-commands.md).

## Project structure

```
.
├── docker-compose.yml     # the stack definition
├── docker-commands.md     # full command reference / runbook
├── .env.example            # environment variable template (safe to commit)
├── .env                     # your real secrets (git-ignored, not committed)
├── .gitignore
├── data/                     # Postgres's persisted data (git-ignored)
└── README.md
```

## Why a healthcheck-gated dependency instead of just `depends_on`?

Plain `depends_on` (without a `condition`) only waits for a
container to *start*, not for the application inside it to be
*ready*. Keycloak in particular takes several seconds after its
process starts before it can actually serve traffic or accept a DB
connection. Gating `curl` on `service_healthy` for both dependencies
means it never starts against a half-initialized stack — the same
pattern you'd want before running migrations, integration tests, or
any dependent service in a real deployment.

## Logging strategy

Docker's default `json-file` driver wraps every log line in a JSON
envelope, which isn't the desired format here. Every service instead
uses the `local` driver — a compact, non-JSON binary format — with
`max-size` and `max-file: "20"` set so old log segments are
automatically dropped once the cap is hit, keeping disk usage bounded
without a separate cron/logrotate job.

## License

MIT — see [LICENSE](./LICENSE).