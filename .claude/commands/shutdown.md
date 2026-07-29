# /shutdown

Stops the web and worker services on Railway. Postgres and Redis are left running.

## Mechanism

Uses `railway down --service <name>` for both services. This removes the active deployment, which is the Level B shutdown mode. Level A (pause deployment while retaining it) is not used.

## Steps

1. Confirm the operator wants to shut down. Ask explicitly.

2. Stop the worker service:

```
   railway down --service worker --yes
```

3. Stop the web service:

```
   railway down --service web --yes
```

4. Verify each service has no active deployment by querying GraphQL:

```
   query {
     service(id: "<service_id>") {
       name
       deployments(first: 1) {
         edges { node { id status deploymentStopped } }
       }
     }
   }
```

   Expect `deployments.edges` to be empty, or the latest deployment to be in a removed state.

5. Verify the public URL returns 404 with `x-railway-fallback: true`:

```
   curl.exe -I https://web-production-35eb3.up.railway.app
```

6. Report the shutdown result per service.

## Why Level B only

Level A shutdowns on this project are unreliable. `railway scale replicas=0` silently no-ops on CLI 5.30.1. The `deploymentStop` GraphQL mutation was observed to succeed on worker but no-op on web in the same session. Level B via `railway down` is the only mechanism that consistently produces a verifiable stopped state on both services.

The tradeoff is that restart requires a rebuild via `railway up` rather than a redeploy of the retained deployment. This is acceptable for this project's cadence.

## Preserved state

`railway down` removes the deployment. It does not remove:

- Environment variables and reference variables
- Domain configuration and target port
- Start command (worker)
- Data in Postgres and Redis

These survive shutdown and are available on restart.
