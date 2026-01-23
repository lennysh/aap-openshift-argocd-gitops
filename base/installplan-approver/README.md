# InstallPlan Approver

This component provides automated approval of OpenShift Operator Lifecycle Manager (OLM) InstallPlans during the initial operator deployment.

## What It Does

The `installplan-approver` is an ArgoCD hook that automatically approves the InstallPlan created when a new operator subscription is established. It consists of:

- **ServiceAccount**: Provides identity for the approval job
- **Role**: Grants permissions to get, list, and patch InstallPlans
- **RoleBinding**: Binds the Role to the ServiceAccount
- **Job**: Executes the approval logic as an ArgoCD Sync hook

## Why We Need It

### Manual InstallPlan Approval

In the operator subscription (`app-aap/03-subscription.yml`), we set:

```yaml
spec:
  installPlanApproval: Manual
```

This configuration ensures that **future operator upgrades require manual approval**, preventing unexpected updates that could disrupt your environment.

### The Challenge

When `installPlanApproval: Manual` is set, the initial InstallPlan (created during first deployment) also requires manual approval. Without intervention, the operator installation will hang waiting for approval, blocking the entire GitOps deployment.

### The Solution

The `installplan-approver` hook automatically approves the **initial** InstallPlan, allowing the deployment to proceed automatically. Future upgrades will still require manual approval, giving you control over when updates happen.

## How It Works

1. **PreSync Phase** (Sync Wave 1):
   - ServiceAccount, Role, and RoleBinding are created
   - These provide the necessary permissions for the job

2. **Sync Phase** (Sync Wave 3):
   - The Subscription is created, which triggers OLM to create an InstallPlan
   - The `installplan-approver` Job runs as a Sync hook in parallel
   - The job script:
     - Waits up to 5 minutes for an InstallPlan to appear
     - Checks if the InstallPlan is already approved (idempotent)
     - Patches the InstallPlan to set `spec.approved: true`
     - Exits successfully

3. **Post-Hook**:
   - ArgoCD automatically deletes the job pod after successful completion (via `hook-delete-policy: HookSucceeded`)
   - The job resource itself is retained for 24 hours (via `ttlSecondsAfterFinished: 86400`)

## Sync Wave Configuration

The job uses sync wave `3`, which matches the Subscription's sync wave. This ensures they run together:

- **Sync Wave 2**: OperatorGroup is created
- **Sync Wave 3**: Subscription is created AND InstallPlan approval hook runs (in parallel)
- **Sync Wave 4**: AnsibleAutomationPlatform instance is created (after operator is installed)

## Security Considerations

The job runs with minimal privileges:
- Non-root user (`runAsNonRoot: true`)
- Dropped capabilities (`drop: ["ALL"]`)
- Read-only root filesystem
- Resource limits applied
- Only has permissions to manage InstallPlans (not other cluster resources)

## Troubleshooting

### Job Fails to Find InstallPlan

If the job times out waiting for an InstallPlan:

1. Check if the Subscription was created:
   ```bash
   oc get subscription -n <namespace>
   ```

2. Check if an InstallPlan exists:
   ```bash
   oc get installplan -n <namespace>
   ```

3. Check job logs:
   ```bash
   oc logs -n <namespace> -l job-name=installplan-approver
   ```

### InstallPlan Already Approved

The job is idempotent - if the InstallPlan is already approved, it will exit successfully without error.

### Job Stuck Running

If the job appears stuck, check:
- Subscription status (may be waiting for catalog source)
- Network connectivity from the job pod
- RBAC permissions (Role and RoleBinding)

## Credits

This component was originally developed by [@CastawayEGR](https://github.com/CastawayEGR) (Mike Tipton) as part of his [aap-ocp-gitops repository](https://github.com/CastawayEGR/aap-ocp-gitops). Special thanks to Mike for sharing this solution and providing invaluable assistance.

## References

- [Operator Lifecycle Manager Documentation](https://olm.operatorframework.io/)
- [ArgoCD Hooks Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks/)
- [InstallPlan API Reference](https://olm.operatorframework.io/docs/concepts/crds/installplan/)
