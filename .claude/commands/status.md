# Status Report

Report the current state of all four Railway services in a single pass. Do not modify anything.

## Execution

1. Fetch status for each service:
   - `railway status --service web`
   - `railway status --service worker`
   - `railway status --service Postgres`
   - `railway status --service Redis`

2. Fetch the last 30 lines of logs for web and worker:
   - `railway logs --service web --deployment | tail -n 30`
   - `railway logs --service worker --deployment | tail -n 30`

3. Test web public URL:
   - `curl -I https://web-production-35eb3.up.railway.app`

4. Report a summary table:

| Service  | Deployment Status | Recent Errors | Notes |
|----------|-------------------|---------------|-------|
| web      |                   |               |       |
| worker   |                   |               |       |
| Postgres |                   |               |       |
| Redis    |                   |               |       |

5. Highlight any of the following:
   - Deployment status other than SUCCESS
   - Redis auth errors (NOAUTH, WRONGPASS)
   - Postgres connection errors (OperationalError, connection refused)
   - Web returning 502 with `x-railway-fallback: true`
   - Worker running `gunicorn` instead of `celery`
   - Any restarts within the last hour

Do not modify anything. Report only.
