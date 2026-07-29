# Shutdown Procedure

Ask the operator which shutdown level applies before executing anything:

- **Level A (Pause):** Services stop running, data and variables retained. Reversible via dashboard resume or `/restart`.
- **Level B (Remove deployments):** Stop compute, retain data services and variables. Reversible via `railway up`.
- **Level C (Delete project):** Permanent. All data, variables, and history lost. Requires explicit confirmation and offer to take a Postgres backup first.

Do NOT proceed without operator confirmation of the level.

## Execution order (all levels)

1. Confirm operator's chosen level. If Level C, offer to run `railway run --service Postgres pg_dump` and save output locally first.

2. Stop worker first (allows in-flight Celery tasks to drain):
   - Level A: pause worker via dashboard or `railway service` UI command
   - Level B: `railway down --service worker`
   - Level C: proceed to project delete after backup

3. Stop web second:
   - Level A: pause web
   - Level B: `railway down --service web`
   - Level C: proceed

4. Postgres and Redis last (or leave running for Level A/B if operator prefers):
   - Level A: pause each
   - Level B: leave running (removing deployments does not affect databases)
   - Level C: project delete removes them

5. Verify final state:
   - `railway status --service worker`
   - `railway status --service web`
   - `railway status --service Postgres`
   - `railway status --service Redis`

6. Report final state to operator.

Do not modify repo files during shutdown.
