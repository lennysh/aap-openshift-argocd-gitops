# Ansible Automation Platform on OpenShift with ArgoCD GitOps

GitOps-managed deployment of [Ansible Automation Platform (AAP)](https://www.redhat.com/en/technologies/management/ansible) on OpenShift using ArgoCD and Kustomize.

## Overview

This repository provides a complete GitOps workflow for deploying Ansible Automation Platform on Red Hat OpenShift. It uses ArgoCD for continuous deployment, Kustomize for configuration management, and follows best practices for operator lifecycle management.

## Features

- **GitOps-driven deployment** using ArgoCD
- **App-of-Apps pattern** - The `cluster-config` Application manages itself and all other applications
- **Automated InstallPlan approval** for initial operator installation
- **Manual InstallPlan approval** for future upgrades (prevents unexpected updates)
- **Kustomize-based configuration** for easy customization
- **Multi-component support** including AAP, DevSpaces, and OPA
- **Sync wave management** for proper resource ordering

## Prerequisites

- Red Hat OpenShift cluster (4.x or later)
- ArgoCD installed (OpenShift GitOps operator)
- `oc` CLI tool configured and authenticated
- Access to `redhat-operators` catalog source
- Permissions to create namespaces and deploy operators

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/lennysh/aap-openshift-argocd-gitops-demo.git
cd aap-openshift-argocd-gitops-demo
```

### 2. Update Repository URL (if forking)

If you're using your own fork, update the repository URL in:
- `cluster/applications/cluster-config.yml` (the app-of-apps)
- `cluster/applications/aap.yml` (and other application files)
- Change `repoURL` to point to your repository

### 3. Customize Configuration

Edit the application manifests in `app-aap/` to match your requirements:
- **Namespace**: Update namespace name in `01-namespace.yml` and `kustomization.yml`
- **Operator Channel**: Set desired AAP version in `03-subscription.yml` (e.g., `stable-2.6`)
- **AAP Instance**: Configure AAP components in `04-aap.yml`

### 4. Deploy to ArgoCD

#### Option A: Bootstrap cluster-config (App-of-Apps)

Create the `cluster-config` Application manually in the ArgoCD UI:
- Point it to your Git repository
- Set the path to `cluster`
- Enable recursive directory scanning

Once created, the `cluster-config` Application will automatically detect and manage all Application YAML files in `cluster/applications/`, including itself!

#### Option B: Manual Application Deployment

Alternatively, you can apply individual Application manifests:

```bash
oc apply -f cluster/applications/aap.yml
```

### 5. Monitor Deployment

Watch the ArgoCD application status:

```bash
# Check cluster-config (app-of-apps)
oc get application cluster-config -n openshift-gitops -w

# Check individual applications
oc get application aap -n openshift-gitops -w
oc get applications -n openshift-gitops
```

Or view in the ArgoCD UI:

```bash
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

## Directory Structure

```
├── app-aap/                   # AAP application manifests
│   ├── 01-namespace.yml       # Namespace definition
│   ├── 02-operatorgroup.yml   # OperatorGroup for OLM
│   ├── 03-subscription.yml    # Operator subscription (Manual approval)
│   ├── 04-aap.yml             # AAP instance configuration
│   └── kustomization.yml      # Kustomize configuration
├── app-aap-opa/               # OPA (Open Policy Agent) deployment
├── app-devspaces/             # DevSpaces configuration
├── base/                      # Shared base components
│   └── installplan-approver/  # Automated InstallPlan approval job
├── cluster/                   # ArgoCD Application definitions
│   └── applications/          # Application manifests for ArgoCD
│       ├── cluster-config.yml # App-of-apps (manages itself and all apps)
│       ├── aap.yml            # AAP Application
│       ├── aap-opa.yml        # OPA Application
│       └── devspaces.yml      # DevSpaces Application
└── README.md                  # This file
```

## How It Works

### App-of-Apps Pattern

This repository uses the **App-of-Apps** pattern:

1. **`cluster-config` Application** (`cluster/applications/cluster-config.yml`):
   - Watches the `cluster/` directory recursively
   - Automatically manages all Application YAML files in `cluster/applications/`
   - **Manages itself** - any changes to `cluster-config.yml` are automatically applied
   - Uses sync wave `0` to ensure it syncs before other applications

2. **Individual Applications** (`cluster/applications/*.yml`):
   - Each Application YAML file defines how to deploy a specific component
   - Automatically discovered and managed by `cluster-config`
   - Changes to any Application YAML are automatically synced by ArgoCD

### Adding New Applications

To add a new application:

1. **Create the Application manually in ArgoCD UI** (to bootstrap):
   - Point it to your Git repository and the path containing your app manifests
   - Use the same name you'll use in the YAML file

2. **Create the Application YAML file** in `cluster/applications/`:
   ```bash
   # Copy an existing app as a template
   cp cluster/applications/aap.yml cluster/applications/my-new-app.yml
   # Edit the file with your app details
   ```

3. **Commit and push**:
   ```bash
   git add cluster/applications/my-new-app.yml
   git commit -m "Add my-new-app Application"
   git push
   ```

4. **`cluster-config` will automatically detect and manage it** going forward

After the YAML is in Git, always make changes via Git, not through the ArgoCD UI.

For detailed instructions on exporting manually created applications, see [cluster/applications/README-export.md](cluster/applications/README-export.md).

### Sync Wave Ordering

Resources are deployed in a specific order using ArgoCD sync waves:

1. **Sync Wave 1**: Namespace creation
2. **Sync Wave 2**: OperatorGroup creation
3. **Sync Wave 3**: Subscription creation + InstallPlan approval hook
4. **Sync Wave 4**: AnsibleAutomationPlatform instance creation

### InstallPlan Approval

The `installplan-approver` component automatically approves the initial InstallPlan when the operator subscription is created. This allows for automated initial deployment while keeping future upgrades manual (see `03-subscription.yml` where `installPlanApproval: Manual`).

For more details, see [base/installplan-approver/README.md](base/installplan-approver/README.md).

## Configuration

### Changing AAP Version

Edit `app-aap/03-subscription.yml`:

```yaml
spec:
  channel: stable-2.6  # Change to desired version channel
```

### Customizing AAP Components

Edit `app-aap/04-aap.yml` to enable/disable or configure:
- Controller
- Event-Driven Ansible (EDA)
- Automation Hub
- Ansible Lightspeed

### Network Policies

The deployment includes network policies for security. Adjust as needed in the respective application directories.

## Troubleshooting

### Application Stuck in Progressing State

Check if the InstallPlan approval job completed:

```bash
oc get job installplan-approver -n <namespace>
oc logs -n <namespace> -l job-name=installplan-approver
```

### Operator Not Installing

Verify the subscription status:

```bash
oc get subscription -n <namespace>
oc get installplan -n <namespace>
oc get csv -n <namespace>
```

### ArgoCD Sync Issues

Check application status and events:

```bash
# Check cluster-config (app-of-apps)
oc describe application cluster-config -n openshift-gitops

# Check individual applications
oc describe application aap -n openshift-gitops
oc get events -n openshift-gitops --sort-by='.lastTimestamp'
```

### Application Not Being Managed

If an application you created manually isn't being managed by Git:

1. Ensure the Application YAML file exists in `cluster/applications/`
2. Verify the name in the YAML matches the manually created Application name
3. Check that `cluster-config` is synced and healthy:
   ```bash
   oc get application cluster-config -n openshift-gitops
   ```
4. The `cluster-config` Application should automatically detect and reconcile the Application

## Additional Applications

This repository also includes configurations for:

- **DevSpaces**: Red Hat OpenShift Dev Spaces for cloud development
- **OPA**: Open Policy Agent for policy enforcement

See their respective directories for configuration details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

Special thanks to [@CastawayEGR](https://github.com/CastawayEGR) (Mike Tipton) and his [aap-ocp-gitops repository](https://github.com/CastawayEGR/aap-ocp-gitops) for the installplan-approver component and invaluable assistance in getting this deployment working.

## Resources

- [Ansible Automation Platform Documentation](https://access.redhat.com/documentation/en-us/ansible_automation_platform/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [OpenShift GitOps](https://docs.openshift.com/container-platform/latest/cicd/gitops/understanding-openshift-gitops.html)
- [Operator Lifecycle Manager](https://olm.operatorframework.io/)
