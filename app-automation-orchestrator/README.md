# Automation Orchestrator — PostgreSQL prerequisites

The Automation Orchestrator operator uses **bring-your-own (BYO) PostgreSQL**. The operator does not create databases or users for you. Complete PostgreSQL setup **before** applying `05-secrets.yml` and `06-automationorchestrator.yml`.

## What you need on PostgreSQL

| Database | Used by | Referenced in |
|---|---|---|
| `backend` (example name) | Application backend | `backend-db-secret` → `spec.postgres.backendDatabase` |
| `temporal` (example name) | Temporal core schema | `temporal-db-secret` → `spec.postgres.temporalDatabase` |
| `temporal_visibility` | Temporal visibility/search schema | **Not** in a secret — hardcoded default used by the migration job |

Each application secret requires these keys:

- `database`
- `username`
- `password`

The PostgreSQL **host** is set on the `AutomationOrchestrator` CR (`spec.postgres.host`), not in secrets.

## Deployment order

1. `01`–`04` — operator and instance namespace
2. **PostgreSQL** — create users and databases (this document)
3. `05-secrets.yml` — cluster-specific credentials (gitignored)
4. `06-automationorchestrator.yml` — `AutomationOrchestrator` CR (managed by Argo CD)

Copy the secrets example before editing credentials:

```bash
cp 05-secrets.example.yml 05-secrets.yml
```

Uncomment `05-secrets.yml` in `kustomization.yml` when ready, or apply secrets with `oc apply -f 05-secrets.yml`.

## Example SQL (manual setup)

Connect as a PostgreSQL superuser (for example `postgres`) and run:

```sql
-- Backend application database
CREATE USER backend WITH PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';
CREATE DATABASE backend OWNER backend;
GRANT ALL PRIVILEGES ON DATABASE backend TO backend;
REVOKE CONNECT ON DATABASE backend FROM PUBLIC;
GRANT CONNECT ON DATABASE backend TO backend;

-- Temporal core database
CREATE USER temporal WITH PASSWORD 'REPLACE_WITH_STRONG_PASSWORD';
CREATE DATABASE temporal OWNER temporal;
GRANT ALL PRIVILEGES ON DATABASE temporal TO temporal;
REVOKE CONNECT ON DATABASE temporal FROM PUBLIC;
GRANT CONNECT ON DATABASE temporal TO temporal;

-- Temporal visibility database (required; uses the temporal user)
CREATE DATABASE temporal_visibility OWNER temporal;
GRANT ALL PRIVILEGES ON DATABASE temporal_visibility TO temporal;
REVOKE CONNECT ON DATABASE temporal_visibility FROM PUBLIC;
GRANT CONNECT ON DATABASE temporal_visibility TO temporal;
```

Verify:

```sql
SELECT datname, pg_catalog.pg_get_userbyid(datdba) AS owner
FROM pg_database
WHERE datname IN ('backend', 'temporal', 'temporal_visibility');
```

Expected: three rows, with `temporal_visibility` owned by `temporal`.

### Connect from your workstation

```bash
psql -h REPLACE_PG_HOST -U postgres -p 5432
```

From OpenShift (when `psql` is not installed locally):

```bash
oc run psql-admin --rm -i --restart=Never -n automation-orchestrator \
  --image=docker.io/postgres:15-alpine \
  --env="PGPASSWORD=REPLACE_POSTGRES_SUPERUSER_PASSWORD" \
  -- psql -h REPLACE_PG_HOST -U postgres -p 5432 -c "\l"
```

## Example: Docker / Proxmox init script pattern

If you provision PostgreSQL with `POSTGRES_CUSTOM_DATABASES` (one user per entry), a typical value is:

```yaml
POSTGRES_CUSTOM_DATABASES: >-
  backend:backend:REPLACE_BACKEND_PASSWORD,
  temporal:temporal:REPLACE_TEMPORAL_PASSWORD
```

That creates `backend` and `temporal` only. **`temporal_visibility` must be added separately**, because the init loop runs `CREATE USER` for each entry and cannot reuse the existing `temporal` user.

Add this after your custom database loop in `init-isolated-databases.sh`:

```bash
if psql -U "$POSTGRES_USER" -d postgres -tAc "SELECT 1 FROM pg_database WHERE datname='temporal'" | grep -q 1 \
   && ! psql -U "$POSTGRES_USER" -d postgres -tAc "SELECT 1 FROM pg_database WHERE datname='temporal_visibility'" | grep -q 1; then
    echo "Creating temporal_visibility database for Temporal..."
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
        CREATE DATABASE temporal_visibility OWNER temporal;
        GRANT ALL PRIVILEGES ON DATABASE temporal_visibility TO temporal;
        REVOKE CONNECT ON DATABASE temporal_visibility FROM PUBLIC;
        GRANT CONNECT ON DATABASE temporal_visibility TO temporal;
EOSQL
fi
```

On an **already running** PostgreSQL instance, run the SQL block under [Example SQL (manual setup)](#example-sql-manual-setup) instead of re-running init.

## Map databases to Kubernetes secrets

Edit `05-secrets.yml` (never commit real values to the public repo):

```yaml
# backend-db-secret
stringData:
  database: backend
  username: backend
  password: REPLACE_BACKEND_PASSWORD

# temporal-db-secret
stringData:
  database: temporal
  username: temporal
  password: REPLACE_TEMPORAL_PASSWORD
```

Apply:

```bash
oc apply -f 05-secrets.yml
```

## Configure the AutomationOrchestrator CR

`06-automationorchestrator.yml` is managed in git and synced by Argo CD. Ensure PostgreSQL and `05-secrets.yml` are in place before the CR reconciles.

Key fields:

```yaml
spec:
  ingress:
    type: Route
    host: automation-orchestrator.apps.example.com
  postgres:
    host: REPLACE_PG_HOST
    port: 5432
    sslMode: prefer   # use require or verify-full if TLS is enforced
    backendDatabase:
      secretRef:
        name: backend-db-secret
    temporalDatabase:
      secretRef:
        name: temporal-db-secret
```

Optional: omit `spec.secrets.initialAdminPasswordSecretRef` to let the operator auto-generate the initial admin password.

## Troubleshooting

### `pq: database "temporal_visibility" does not exist`

The Temporal migration job succeeded on the main `temporal` database but failed on the visibility database. Create `temporal_visibility` (see SQL above), then delete the failed migration job or re-sync the CR:

```bash
oc delete job -n automation-orchestrator -l app.kubernetes.io/component=temporal-migration
```

### Backend migration succeeds, Temporal migration fails

Usually means `backend` and `temporal` exist but `temporal_visibility` does not. All three must exist before the CR reconciles cleanly.

### Public git repository

Keep `05-secrets.yml` out of git (see `.gitignore`). The public repo ships `05-secrets.example.yml` for credentials and manages `06-automationorchestrator.yml` via Argo CD.
