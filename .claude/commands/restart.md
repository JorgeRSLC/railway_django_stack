# Restart Procedure

Determine restart context first:

- **Live restart (no code change):** Use `railway redeploy --service <name>` per service.
- **Restart after Level A pause:** Use dashboard resume or `railway service` resume commands.
- **Restart after Level B removal:** Use `railway up --service <name>` per service.
- **Restart after code changes:** Same as Level B, but ensure changes are committed and pushed to `main` first.

## Execution order

1. Confirm restart context with operator.

2. Ensure Postgres and Redis are running. If paused, resume first. Verify:
   - `railway status --service Postgres`
   - `railway status --service Redis`

3. Restart / redeploy web service:
   - Live: `railway redeploy --service web`
   - After Level B: `railway up --service web --detach`
   Wait 60 seconds, then check `railway status --service web`.

4. Restart / redeploy worker service:
   - Live: `railway redeploy --service worker`
   - After Level B: `railway up --service worker --detach`
   Wait 60 seconds, then check `railway status --service worker`.

5. Verify web reachability:
   - `curl -I https://web-production-35eb3.up.railway.app`
   - Expected: HTTP status 200, 301, 302, or 404 (Django-generated).
   - If 502 with `x-railway-fallback: true`, verify domain target port is 8000 via GraphQL:
     `query { serviceInstance(serviceId: "c0edfdd1-b351-40c9-9745-c514cc00817c", environmentId: "555ff3e9-fabb-47b8-91e7-71c233f58b4c") { domains { targetPort } } }`

6. Verify Celery worker:
   - `railway logs --service worker --deployment | tail -n 50`
   - Expected: `celery@... ready`, `Connected to redis://default:**@redis.railway.internal:6379//`.
   - No `gunicorn` output (would indicate startCommand was lost; re-apply via `serviceInstanceUpdate` mutation).

7. Verify variable references still resolve:
   - `railway variables --service web`
   - `railway variables --service worker`
   - Confirm PGHOST, REDISHOST, REDISUSER, REDISPASSWORD have valid values.

8. Report final state to operator.

Do not modify repo files during restart unless code changes were the trigger.
