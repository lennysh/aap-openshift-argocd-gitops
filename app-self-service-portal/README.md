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

`values.yaml` is referenced by the Helm Application via `$values/values.yaml`. Key fields:

1. `redhat-developer-hub.global.clusterRouterBase` — cluster router base (no `https://`)
2. `catalog.providers.rhaap.production.orgs` — AAP organization name(s)
3. `imageTagInfo` / chart `targetRevision` in `cluster/applications/self-service-portal.yml` per the [lifecycle page](https://access.redhat.com/page/ansible-automation-platform-self-service-automation-portal-lifecycle)

## After deployment

1. Open the portal route and set the OAuth app **Redirect URI** to  
   `https://<portal-host>/api/auth/rhaap/handler/frame`
2. Configure portal RBAC (Administration → RBAC) per the install guide.

## Public git repository

Keep `02-secrets.yml` out of git (see `.gitignore`). The public repo ships `02-secrets.example.yml` for structure; apply real credentials with `oc apply -f 02-secrets.yml` before syncing the Helm app.

## Cutover from a manual Helm install

If you previously deployed with tarball plug-ins and a local `plugin-registry` BuildConfig/ImageStream, that stack is **not** needed for GitOps. This repo uses OCI plug-in delivery instead. Before deleting the old namespace, export values from `secrets-rhaap-portal` and recreate them in `02-secrets.yml`.
