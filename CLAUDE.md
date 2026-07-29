# Railway Django Stack - Project Context

This repository is deployed on Railway with four services. This file provides persistent context for Claude Code sessions.

## Deployment Reference

- Full deployment guide: see `railway_django_deployment.md` (external, not in repo)
- Deployed on Railway project ID: `a005e7da-a3a3-4cda-befe-54e64cb05c7a`
- Environment: `production` (ID: `555ff3e9-fabb-47b8-91e7-71c233f58b4c`)
- Workspace: `growth-labs`

## Services

| Service  | ID                                     | Purpose                           |
|----------|----------------------------------------|-----------------------------------|
| web      | `c0edfdd1-b351-40c9-9745-c514cc00817c` | Django + Gunicorn (public)        |
| worker   | `3300d954-e4df-4857-b166-cf4ac3c75209` | Celery worker + beat scheduler    |
| Postgres | (Railway-managed)                      | Primary database                  |
| Redis    | (Railway-managed)                      | Celery broker + Django cache      |

Public URL: `https://web-production-35eb3.up.railway.app`

## Operational Constraints

1. **Line endings:** `.gitattributes` enforces LF on `.sh`, `.py`, and `Dockerfile`. Do not disable or remove. CRLF on shell scripts causes container exec failures.

2. **Worker start command:** Set via Railway GraphQL API (not in repo, not in dashboard-accessible form). Value: `celery -A railway_django_stack worker --beat -l INFO`. If the worker service is deleted and recreated, this must be re-applied via the `serviceInstanceUpdate` mutation.

3. **Web domain target port:** Set explicitly to 8000 via GraphQL API because `deployment/server-entrypoint.sh` hardcodes `--bind 0.0.0.0:8000` instead of reading `$PORT`. If a new domain is created, target port must be set again.

4. **Redis authentication:** Web and worker services require `REDISUSER` and `REDISPASSWORD` env vars (set as reference variables to the Redis service). The app code reads these to construct the authenticated Redis URL.

5. **RUN_MIGRATIONS=True** on the web service triggers `manage.py migrate` on container startup via the entrypoint script. Migrations are idempotent.

## Available Slash Commands

- `/shutdown` - Stop web and worker via `railway down` (Level B). Postgres and Redis are left running.
- `/restart` - Restart web and worker, choosing `railway up` or `railway redeploy` per service based on GraphQL-queried state.
- `/status` - Report current state of all four services via GraphQL. Read-only.

## Local Development

The repo includes `docker-compose.yml` mirroring the Railway topology. Run `docker-compose up --build` from the repo root for local testing before pushing.

## Known Follow-ups

1. Remove redundant `RAILWAY_RUN_COMMAND` variable from worker service.
2. Rotate Postgres password (exposed during initial provisioning).
3. Modify `deployment/server-entrypoint.sh` to bind Gunicorn to `$PORT` with fallback to 8000.
4. Verify healthcheck endpoint at `/healthcheck/`.

## Railway Platform Gotchas

Observed behaviors of the Railway CLI (5.30.1) and GraphQL API on this project. These constrain how shutdown, restart, and status commands are implemented.

1. **`railway scale replicas=0` silently no-ops.** On CLI 5.30.1, the command returns success but does not scale the service. Do not use for shutdowns.

2. **`deploymentStop` GraphQL mutation is inconsistent.** In one observed session, the mutation succeeded on the worker service and no-op'd on the web service without an error. Do not rely on it as a shutdown mechanism. Use `railway down --service <name>` instead.

3. **Verify shutdown effectiveness by querying `latestDeployment` via GraphQL.** CLI return codes do not reliably reflect deployment state after shutdown attempts. Query the service's latest deployment and inspect `deploymentStopped` or the absence of deployments in `edges` to confirm the stopped state.

4. **`railway status --service <name>` is not supported on CLI 5.30.1.** The `--service` flag is rejected. For per-service status, use the GraphQL `service` query with a `deployments(first: 1)` selection instead.
