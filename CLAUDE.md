# CLAUDE.md

This file provides guidance to Claude Code when working with this GitOps repository.

## Project Overview

This is a **GitOps repository** for managing Kubernetes/OpenShift infrastructure and applications using **ArgoCD**. The repository follows a structured approach with Helm charts for templating and ArgoCD Applications for deployment automation.

## Repository Structure

```
my-gitops/
├── gitops/                          # ArgoCD Application definitions
│   ├── infra/                       # Infrastructure applications
│   │   ├── application-infra.yaml   # Main infra app (Helm + OpenBao)
│   │   └── application-openbao.yaml # OpenBao vault application
│   └── applications/                # Application deployments
│       ├── helloworld-ear/
│       └── honeybees-ear/
├── manifests/                       # Kubernetes manifests
│   ├── helm/                        # Helm charts
│   │   └── infra/                   # Infrastructure Helm chart
│   │       ├── Chart.yaml
│   │       ├── values.yaml          # Default values
│   │       └── templates/           # Helm templates
│   │           ├── namespaces.yaml
│   │           ├── rbac.yaml
│   │           ├── argocds.yaml
│   │           ├── app-projects.yaml
│   │           ├── operator-groups.yaml
│   │           ├── subscriptions.yaml
│   │           ├── postgres.yaml
│   │           ├── vms.yaml
│   │           └── console-plugins.yaml
│   ├── kustomize/                   # Kustomize overlays
│   └── plain/                       # Plain YAML manifests
└── eap/                             # EAP (Enterprise Application Platform) configs
```

## Key Conventions

### ArgoCD Sync Waves

Infrastructure resources are deployed in a specific order using ArgoCD sync waves:

- **Wave 0**: RBAC (Groups, RoleBindings, ClusterRoleBindings) + Core namespaces + ArgoCD instances
- **Wave 1**: (Reserved for future use)
- **Wave 2**: Application Projects (AppProjects)
- **Wave 3**: Operators (OperatorGroups, Subscriptions)
- **Wave 4**: Application resources (StatefulSets, Services)
- **Wave 5**: (Reserved for future use)
- **Wave 6**: Load-balanced services

**Important**: Always add `argocd.argoproj.io/sync-wave` annotations to new resources to ensure proper deployment order.

### Helm Templating

The `manifests/helm/infra` chart uses Helm templating to make infrastructure configurable:

1. **values.yaml**: Contains default configuration
2. **ArgoCD Application parameters**: Override values.yaml settings via `helm.parameters` in Application manifests

#### Parameterized Resources

The following resources are configured via `values.yaml`:

- **Namespaces**: Dynamic list in `namespaces` array (application namespaces only)
  - Hardcoded namespaces: `external-secrets-operator`, `openshift-logging`, `hello-world-logging`, `openbao`, `openshift-gitops-operator`
- **AppProject destinations**: `appProjects.sharedAppProject.destinations`
- **AppProject namespace blacklist**: `appProjects.sharedAppProject.namespaceResourceBlacklist`
- **AppProject source repos**: `appProjects.sharedAppProject.sourceRepos`
- **Developer groups**: `appProjects.sharedAppProject.roles.developerGroups`

#### Passing Parameters from ArgoCD

In `gitops/infra/application-infra.yaml`, parameters are passed as arrays:

```yaml
helm:
  parameters:
    - name: namespaces[0]
      value: hello-world-s2i
    - name: namespaces[1]
      value: hello-world-deploy
```

### Namespace Management

Two types of namespaces:

1. **Infrastructure namespaces** (hardcoded in `templates/namespaces.yaml`):
   - `external-secrets-operator`
   - `openshift-logging`
   - `hello-world-logging`
   - `openbao`
   - `openshift-gitops-operator`

2. **Application namespaces** (configurable via values.yaml):
   - Defined in `values.yaml` under `namespaces` array
   - Overridden by ArgoCD Application parameters
   - Examples: `hello-world-s2i`, `hello-world-deploy`, `honey-bees-s2i`, `honey-bees-deploy`

## Working with This Repository

### Adding a New Namespace

1. Add to ArgoCD Application parameters in `gitops/infra/application-infra.yaml`:
```yaml
- name: namespaces[N]
  value: new-namespace-name
```

2. The namespace will be created with sync wave "0"

### Adding a New AppProject Destination

1. Edit `manifests/helm/infra/values.yaml`:
```yaml
appProjects:
  sharedAppProject:
    destinations:
      - namespace: 'new-namespace'
        server: 'https://kubernetes.default.svc'
```

### Adding a New Operator

1. Create OperatorGroup if needed in `templates/operator-groups.yaml`
2. Create Subscription in `templates/subscriptions.yaml`
3. Add sync wave "3" annotation to both
4. Ensure the target namespace exists (add to namespaces if needed)

### Modifying Sync Waves

When changing sync waves, consider dependencies:
- Namespaces must exist before resources in those namespaces
- RBAC should be established early
- Operators need namespaces and operator groups
- Applications need operators to be ready

## ArgoCD Application Structure

### Multi-Source Applications

`application-infra.yaml` uses multiple sources:
1. **OpenBao Helm chart** from upstream repository
2. **Local Helm chart** from this repository (`manifests/helm/infra`)

Each source can have its own configuration but shares the same sync wave.

### Automated Sync Policy

Applications use automated sync with:
- `prune: true` - Remove resources not in Git
- `selfHeal: true` - Revert manual changes
- `SkipDryRunOnMissingResource=true` - Skip validation for CRDs not yet installed

## Important Files

- **gitops/infra/application-infra.yaml**: Main infrastructure application
- **manifests/helm/infra/values.yaml**: Default configuration values
- **manifests/helm/infra/templates/**: All Kubernetes resource templates
- **manifests/helm/infra/Chart.yaml**: Helm chart metadata

## Best Practices

1. **Always use templates**: Don't hardcode values that might change per environment
2. **Use sync waves**: Ensure proper resource ordering
3. **Parameterize via ArgoCD**: Override Helm values through Application parameters, not by modifying values.yaml directly
4. **Test changes**: Verify Helm rendering with `helm template` before committing
5. **Document sync wave changes**: Update this file when adding new resource types
6. **Follow namespace patterns**: Infrastructure namespaces are hardcoded, application namespaces are parameterized

## Common Commands

```bash
# Render Helm chart locally
helm template manifests/helm/infra

# Render with custom values
helm template manifests/helm/infra --set namespaces[0]=test-namespace

# Validate ArgoCD application
kubectl apply --dry-run=client -f gitops/infra/application-infra.yaml
```
