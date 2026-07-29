# /restart

Restarts the web and worker services on Railway. Handles the case where the two services may be in different stopped states.

## Mechanism

Queries each service's `latestDeployment` via GraphQL. Chooses the restart mechanism per service based on the result:

- If no deployment exists (Level B removal): `railway up --service <name> --detach` rebuilds from the local working tree.
- If a deployment exists with `deploymentStopped=true` (Level A stop): `railway redeploy --service <name>` redeploys the retained build.
- If a deployment exists and is running: skip the service.

Reports the mechanism used per service.

## Prerequisites

- Local working tree at `C:\Dev\railway_django_stack` is on the intended commit.
- Railway CLI is authenticated and linked to the `railway_django_stack` project.
- Project access token is available for GraphQL queries.

## Steps

1. Confirm the operator wants to restart. Ask explicitly.

2. For each service (web, then worker), query the latest deployment:

```
   query {
     service(id: "<service_id>") {
       name
       deployments(first: 1) {
         edges { node { id status deploymentStopped canRedeploy } }
       }
     }
   }
```

3. Determine the restart mechanism per service:

   - Empty `edges` array: use `railway up`.
   - Non-empty with `deploymentStopped=true`: use `railway redeploy`.
   - Non-empty with `deploymentStopped=false` and a running status: skip.

4. Execute the chosen mechanism per service. Web first, worker second (matches the migration ordering rule).

   Rebuild path:

```
   railway up --service <name> --detach
```

   Redeploy path:

```
   railway redeploy --service <name> --yes
```

5. Verify each service reaches a running state. Query GraphQL again and confirm the latest deployment is `SUCCESS` with `deploymentStopped=false`.

   For web, additionally verify the public URL no longer returns the Railway edge fallback. The distinguishing signal is the `x-railway-fallback: true` header, not the status code:

```
   curl.exe -I https://web-production-35eb3.up.railway.app
```

   Expect the header to be absent. A 404 without the fallback header indicates a running Django app with no route at `/` and is acceptable.

   - Worker verification: check recent logs for the Celery ready banner and confirm no Gunicorn output:

```
     railway logs --service worker --deployment | Select-Object -Last 100
```

     Expect `celery@... ready`. Gunicorn lines indicate the start command was not applied and the worker booted as web.

   - Reference variable resolution: confirm `PGHOST`, `REDISHOST`, `REDISUSER`, and `REDISPASSWORD` resolve to non-empty values on both services:

```
     railway variables --service web
     railway variables --service worker
```

     Empty or literal `${{...}}` values indicate a stale reference, usually from a recreated Postgres or Redis service.

6. Report per service: the mechanism used, the final deployment status, and any anomalies.

## Ordering

Web restarts before worker. This matches the migration ordering rule from `CLAUDE.md` §5: the web entrypoint runs `manage.py migrate` on startup, and the worker should not run against a schema that has not yet been migrated.

## Failure handling

If a `railway up` build fails, report the build log location and stop. Do not attempt to restart the other service.

If a `railway redeploy` fails with an error indicating the retained deployment is no longer valid, fall back to `railway up --service <name> --detach` and report the fallback.
