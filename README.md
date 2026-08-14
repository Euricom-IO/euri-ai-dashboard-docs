# Euri AI Ops — Docker images

Container images for **Euri AI Ops**, an observability dashboard for Claude
Code and GitHub Copilot usage.

Two images are published:

| Image | Purpose | Default port |
| --- | --- | --- |
| `ghcr.io/euricom-io/ai-ops-dashboard` | Web dashboard (TanStack Start) | `3000` |
| `ghcr.io/euricom-io/ai-ops-otel-server` | OTLP ingest server (Hono + gRPC) | `4318` |

Both require a Postgres database. The `ai-ops-otel-server` image runs the
database schema migrations automatically on startup — no separate migration
step needed.

## Quick start (docker compose)

```yaml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: claude
      POSTGRES_PASSWORD: claude
      POSTGRES_DB: claude_dashboard
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U claude -d claude_dashboard"]
      interval: 5s
      timeout: 5s
      retries: 10

  dashboard:
    image: ghcr.io/euricom-io/ai-ops-dashboard:latest
    ports:
      - "3000:3000"
    environment:
      PORT: 3000
      DATABASE_URL: postgresql://claude:claude@postgres:5432/claude_dashboard
      BETTER_AUTH_SECRET: <random 32+ char secret, e.g. `openssl rand -hex 32`>
      BETTER_AUTH_URL: http://localhost:3000
      APP_URL: http://localhost:3000
      SENDGRID_API_KEY: <your SendGrid key>
    depends_on:
      postgres:
        condition: service_healthy

  otel-server:
    image: ghcr.io/euricom-io/ai-ops-otel-server:latest
    ports:
      - "4318:4318"
    environment:
      PORT: 4318
      DATABASE_URL: postgresql://claude:claude@postgres:5432/claude_dashboard
      API_TOKEN: <bearer token required on incoming OTLP requests>
    depends_on:
      postgres:
        condition: service_healthy
```

Run with `docker compose up -d`, then open http://localhost:3000.

## Configuration

### `ai-ops-dashboard`

| Variable | Required | Notes |
| --- | --- | --- |
| `DATABASE_URL` | yes | Postgres connection string |
| `BETTER_AUTH_SECRET` | yes | random secret, e.g. `openssl rand -hex 32` |
| `BETTER_AUTH_URL` | yes | public base URL of the dashboard |
| `APP_URL` | yes | used in emails/links sent by the dashboard |
| `SENDGRID_API_KEY` | yes | needed to send login/OTP emails |
| `SENDGRID_FROM_EMAIL` | no | default `noreply@euri.com` |
| `BOOTSTRAP_ADMIN_DOMAIN` | no | email domain allowed to self-register on an empty database; the first user for it becomes global admin — unset once bootstrapped |
| `PORT` | no | default `3000` |
| `LOG_LEVEL` | no | pino level, default `info` |
| `FEATURE_EXTENDED_PERIOD` | no | feature flag, default off |
| `SUBSCRIPTION_STANDARD_EUR` / `SUBSCRIPTION_PREMIUM_EUR` | no | cost projection inputs |

### `ai-ops-otel-server`

| Variable | Required | Notes |
| --- | --- | --- |
| `DATABASE_URL` | yes | Postgres connection string |
| `API_TOKEN` | yes | bearer token required on incoming OTLP requests (`Authorization: Bearer <API_TOKEN>`) |
| `PORT` | no | HTTP ingest port, default `4318` |
| `LOG_LEVEL` | no | pino level, default `info` |
| `GITHUB_TOKEN` | only for the Copilot backfill script | GitHub PAT with org Copilot metrics/billing read access |

On startup, `ai-ops-otel-server` runs the Postgres schema migrations
automatically — no separate migration step needed.

## Sending telemetry

Point Claude Code's OTLP exporter at `http://<otel-server-host>:4318`, with
the configured `API_TOKEN` as a bearer token. Add this to `.claude/settings.json`
(user, project, or managed — see [Claude Code settings](https://code.claude.com/docs/en/settings)):

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/json",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://<otel-server-host>:4318",
    "OTEL_EXPORTER_OTLP_HEADERS": "Authorization=Bearer <API_TOKEN>",
    "OTEL_RESOURCE_ATTRIBUTES": "department=engineering,cost_center=eng-123"
  }
}
```

`OTEL_RESOURCE_ATTRIBUTES` is optional — any `key=value` pairs you add there
show up as extra attributes on the ingested telemetry.

See the main repo's
[docs/environment.md](https://github.com/Euricom-IO/euri-ai-dashboard/blob/main/docs/environment.md)
and [docs/otel-server-github.md](https://github.com/Euricom-IO/euri-ai-dashboard/blob/main/docs/otel-server-github.md)
for exporter configuration details.

## Tags

Images are tagged with the release version (e.g. `0.3.0`) and `latest`.

## Support

Issues and source: https://github.com/Euricom-IO/euri-ai-dashboard
