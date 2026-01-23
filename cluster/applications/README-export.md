# Adding New Applications to Git Management

This directory contains ArgoCD Application manifests that are automatically managed by the `cluster-config` Application (app-of-apps pattern).

## Quick Method: Copy Template

The easiest way to add a new application:

1. **Copy an existing Application as a template:**
   ```bash
   cp cluster/applications/aap.yml cluster/applications/my-new-app.yml
   ```

2. **Edit the file** and update:
   - `metadata.name` - Your application name
   - `spec.source.path` - Path to your app manifests in the repo
   - `spec.destination.namespace` - Target namespace
   - `argocd.argoproj.io/sync-wave` annotation if needed

3. **Create the Application manually in ArgoCD UI** (to bootstrap):
   - Use the same name as in the YAML file
   - Point to your Git repo and app manifest path

4. **Commit and push:**
   ```bash
   git add cluster/applications/my-new-app.yml
   git commit -m "Add my-new-app Application"
   git push
   ```

5. **`cluster-config` will automatically detect and manage it** going forward

## Alternative: Export from Manually Created App

If you've already created an app manually and want to manage it via Git:

### 1. Export the Application

```bash
oc get application <app-name> -n openshift-gitops -o yaml > cluster/applications/<app-name>.yml
```

### 2. Clean Up the Exported YAML

Remove these runtime fields that shouldn't be in Git:

- **Remove the entire `status:` section**
- **Remove the entire `operation:` section** (if present)
- Remove `metadata.uid`
- Remove `metadata.resourceVersion`
- Remove `metadata.creationTimestamp`
- Remove `metadata.generation`
- Remove `metadata.managedFields`
- Remove the `argocd.argoproj.io/tracking-id` annotation (if present)

### 3. Keep Only These Fields

Your final YAML should match the format of other files in this directory:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: openshift-gitops
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Adjust if needed
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: <your-repo-url>
    path: <path-to-app-manifests>
    targetRevision: HEAD
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: <target-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - SkipDryRunOnMissingResource=true
      - RespectIgnoreDifferences=true
      - ServerSideApply=true
    retry:
      limit: 10
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 5m
```

### 4. Commit and Push

```bash
git add cluster/applications/<app-name>.yml
git commit -m "Add <app-name> Application to Git management"
git push
```

### 5. Let cluster-config Take Over

Once the YAML file is in Git and committed:

- The `cluster-config` Application (which watches `cluster/` recursively) will automatically detect the new Application
- ArgoCD will reconcile the manually created app with the Git version
- Going forward, all changes should be made to the YAML file in Git, not through the UI

## Important Notes

- The `cluster-config` Application uses `recurse: true` on the `cluster/` path, so it automatically picks up any new Application YAML files
- After the YAML is in Git, **always make changes via Git**, not the ArgoCD UI
- The Application will be automatically synced by `cluster-config` whenever you push changes
- The Application name in the YAML **must match** the name you use when creating it manually (if using the export method)
