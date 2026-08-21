# Self-service automation portal (GitOps)

Deploys the [self-service automation portal](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/installing_self-service_automation_portal) Helm chart (`redhat-rhaap-portal`) into namespace `self-service-portal`.

Argo CD applications:

| Application | Purpose |
|-------------|---------|
| `self-service-portal-prereqs` | Namespace |
| `self-service-portal` | Helm release from `https://charts.openshift.io/` |

## Deployment order

1. `01-namespace.yml` — sync `self-service-portal-prereqs`
2. **AAP prerequisites** — OAuth app, API token, platform gateway setting (install guide)
3. `02-secrets.yml` — cluster-specific credentials (gitignored)
4. `values.yaml` — cluster router base, orgs, RBAC (managed in git)
5. Sync `self-service-portal` Helm Application

Copy the secrets example before editing credentials:

```bash
cp 02-secrets.example.yml 02-secrets.yml
```

Uncomment `02-secrets.yml` in `kustomization.yml` when ready, or apply secrets with `oc apply -f 02-secrets.yml`.

## Required secrets (`02-secrets.yml`)

Apply **before** the Helm Application syncs. Exact secret names and keys are required by the chart.

### `secrets-rhaap-portal`

| Key | Description |
|-----|-------------|
| `aap-host-url` | AAP instance URL (e.g. gateway route for your AAP deployment) |
| `oauth-client-id` | OAuth application client ID |
| `oauth-client-secret` | OAuth application client secret |
| `aap-token` | AAP API token (Write scope) tied to the OAuth app |

### `redhat-rhaap-portal-dynamic-plugins-registry-auth`

Required for OCI plug-in delivery (`pluginMode: oci` in `values.yaml`). Use a [Registry Service Account](https://access.redhat.com/RegistryAuthentication) (not your personal Red Hat password). Secret name must match the Helm release name (`redhat-rhaap-portal`).

Generate `auth.json` from your RSA, then either paste into `02-secrets.yml` or create directly:

```bash
oc create secret generic redhat-rhaap-portal-dynamic-plugins-registry-auth -n self-service-portal \
  --from-file=auth.json=./auth.json
```

### `secrets-scm` (optional)

Only if you use private GitHub/GitLab repos for custom templates.

## Configure Git values

`values.yaml` is referenced by the Helm Application via `$values/app-self-service-portal/values.yaml`. Key fields:

1. `redhat-developer-hub.global.clusterRouterBase` — cluster router base (no `https://`)
2. AAP org sync uses chart defaults (`orgs: Default` under `catalog.providers.rhaap`); do not add a duplicate `production:` key in values — see comment in `values.yaml`
3. `imageTagInfo` / chart `targetRevision` in `cluster/applications/self-service-portal.yml` per the [lifecycle page](https://access.redhat.com/page/ansible-automation-platform-self-service-automation-portal-lifecycle)

### Helm list overrides (RHIDP-6082)

Never partially override chart **lists** in `values.yaml` — Helm replaces the entire list:

| Key | Effect if partially overridden |
|-----|--------------------------------|
| `extraEnvVars` | Drops `POSTGRESQL_ADMIN_PASSWORD`, AAP/OAuth env → backend crash |
| `extraVolumes` | Drops plugin/registry volumes → broken init |
| `catalog.providers` | Duplicate YAML key → backend crash |

### OCI plugin init (~3 min per new pod)

The chart uses an **ephemeral** `volumeClaimTemplate` for `dynamic-plugins-root` — each new pod gets its own 5Gi PVC and downloads OCI plugins (~3 min). That is chart-default and the most GitOps-stable option.

Operational rules:

- **One sync at a time** — let Argo finish before pushing another values change.
- **Do not** `oc rollout restart` — stacked rollouts leave duplicate pods and orphaned plugin PVCs.
- After a rollout settles, prune unused `*-dynamic-plugins-root` PVCs (keep the one bound to the running pod).

### Postgres auth failures (`password authentication failed for user "postgres"`)

The backend reads `${POSTGRESQL_ADMIN_PASSWORD}` from the Helm-managed secret
`redhat-rhaap-portal-postgresql` key `postgres-password`. If many failed rollouts
left the embedded Postgres data dir initialized with a different password, auth
will fail even though the Deployment env vars look correct.

**Fix:** scale the portal Deployment to 0, delete the Postgres pod and its data
PVC (`data-redhat-rhaap-portal-postgresql-0`), wait for Postgres to recreate,
then scale the portal back to 1. On a fresh namespace this should not occur.

### AAP OAuth login fails (`fetch failed` on `/o/token/`)

The portal backend POSTs to `${AAP_HOST_URL}/o/token/` during login. On clusters
using the **default self-signed OpenShift router certificate**, Node.js
`fetch()` rejects TLS (`SSL certificate problem: self-signed certificate in
certificate chain`).

This repo sets `NODE_TLS_REJECT_UNAUTHORIZED=0` via ConfigMap `portal-backend-env`
(`04-portal-backend-env.yml`) — **lab/playground only**. Production should mount
the ingress/router CA and set `NODE_EXTRA_CA_CERTS` instead.

Also confirm the AAP OAuth app **Redirect URI** matches the new portal route:

`https://redhat-rhaap-portal-self-service-portal.apps.ocp001.lennysh.net/api/auth/rhaap/handler/frame`

## After deployment

1. Open the portal route and set the OAuth app **Redirect URI** to  
   `https://<portal-host>/api/auth/rhaap/handler/frame`
2. Configure portal RBAC (Administration → RBAC) per the install guide.

## Public git repository

Keep `02-secrets.yml` out of git (see `.gitignore`). The public repo ships `02-secrets.example.yml` for structure; apply real credentials with `oc apply -f 02-secrets.yml` before syncing the Helm app.

## Cutover from a manual Helm install

If you previously deployed with tarball plug-ins and a local `plugin-registry` BuildConfig/ImageStream, that stack is **not** needed for GitOps. This repo uses OCI plug-in delivery instead. Before deleting the old namespace, export values from `secrets-rhaap-portal` and recreate them in `02-secrets.yml`.
