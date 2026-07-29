# Status Report

Report the current state of all four Railway services in a single pass. Do not modify anything.

## Execution

1. Fetch latest deployment for each service via GraphQL. Repeat this query with the service ID for web, worker, Postgres, and Redis:

```
   query {
     service(id: "<service_id>") {
       name
       deployments(first: 1) {
         edges { node { id status deploymentStopped createdAt } }
       }
     }
   }
```

   Service IDs:
   - web: `c0edfdd1-b351-40c9-9745-c514cc00817c`
   - worker: `3300d954-e4df-4857-b166-cf4ac3c75209`
   - Postgres and Redis: look up via `railway service` or the project query if IDs are not cached.

2. Fetch the last 30 lines of logs for web and worker:

```
   railway logs --service web --deployment | Select-Object -Last 30
   railway logs --service worker --deployment | Select-Object -Last 30
```

3. Test the web public URL:

```
   curl.exe -I https://web-production-35eb3.up.railway.app
```

4. Report a summary table:

   | Service  | Deployment Status | Recent Errors | Notes |
   |----------|-------------------|---------------|-------|
   | web      |                   |               |       |
   | worker   |                   |               |       |
   | Postgres |                   |               |       |
   | Redis    |                   |               |       |

5. Highlight any of the following:

   - Deployment status other than `SUCCESS` with `deploymentStopped=false`.
   - Empty `deployments.edges` (Level B stopped state).
   - `deploymentStopped=true` on a service that should be running (Level A stopped state).
   - Redis auth errors in logs (`NOAUTH`, `WRONGPASS`).
   - Postgres connection errors in logs (`OperationalError`, `connection refused`).
   - Web response including the `x-railway-fallback: true` header, which indicates no active deployment to route to. A 404 without this header is a running Django app with no route at `/` and is acceptable.
   - Worker logs showing `gunicorn` instead of `celery@... ready`, which indicates the start command was not applied.
   - Any restarts within the last hour.

Do not modify anything. Report only.
