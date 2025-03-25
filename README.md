# ArgoCD Infrastructure Template Repository

This repository serves as a template for infrastructure components that are typically referenced from an ArgoCD applications repository. It contains configurations for essential infrastructure components using the App of Apps pattern in ArgoCD.

## Overview

This repository is designed to be used in conjunction with an ArgoCD applications repository, not as a standalone deployment. It uses Helm charts to manage the deployment of various infrastructure components.

The infrastructure components included are:

- **ArgoCD**: Declarative GitOps CD tool for Kubernetes
- **Linkerd**: Ultralight service mesh for Kubernetes
  - Linkerd CRDs
  - Linkerd Control Plane
- **Sealed Secrets**: Controller and tools for one-way encrypted Kubernetes Secrets

## Repository Structure

```
.
├── Chart.yaml              # Helm chart metadata
├── values.yaml            # Global configuration values
├── templates/             # Helm templates
└── infrastructure/        # Infrastructure components
    ├── argocd/           # ArgoCD configuration
    ├── linkerd-crds/     # Linkerd Custom Resource Definitions
    └── linkerd/          # Linkerd service mesh configuration
```

## Configuration

The repository uses a Helm chart named `app-of-apps-infrastructure` to manage all components. Configuration is primarily done through the `values.yaml` file, which includes:

- Individual component configurations
- Repository URLs and versions
- Sync policies and automation settings

## Usage

### Deployment Methods

1. **As Part of ArgoCD Apps Repository**:
   - Reference it from your ArgoCD applications repository
   - Configure the `values.yaml` inside the applications repository `.Values.infrastructure` to match your environment

## Configuration

### Global Settings

The `values.yaml` file contains global settings that apply to all applications:

```yaml
global:
  repoURL: <your-repo-url>
  targetRevision: HEAD
  project: default
  destination:
    server: https://kubernetes.default.svc
```

### Component-Specific Settings

Each infrastructure component can be enabled/disabled and configured independently in the `values.yaml` file.

## Technical Details

### Application Template Structure (templates/applications.yaml)

The `applications.yaml` template is the core of this repository's App of Apps pattern implementation. It generates ArgoCD Application resources using a templating structure that supports both single and multi-source configurations.

#### Root Application

```yaml
{{- if .Values.infrastructure.useInfrastructureAppOfAppsFromInfraRepo }}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps-infrastructure
  namespace: argocd
# ... configuration ...
{{- end }}
```

The root application is conditionally created based on the `useInfrastructureAppOfAppsFromInfraRepo` flag. This allows for deployment of the infrastructure components as part of the applications repository or as a standalone deployment.

#### Child Applications Generation

The template iterates through infrastructure components defined in `.Values.infrastructure`, creating individual ArgoCD Applications for each enabled component:

##### Sync Wave Prioritization
```yaml
annotations:
  {{- if or (eq $appName "argocd") (eq $appName "linkerd-crds") }}
    argocd.argoproj.io/sync-wave: "-10"  # Highest priority
  {{- else if eq $appName "linkerd-control-plane" }}
    argocd.argoproj.io/sync-wave: "-9"   # Second priority
  {{- else }}
    argocd.argoproj.io/sync-wave: "-8"   # Default priority
  {{- end }}
```

Critical components are deployed in a specific order using sync waves:
- Wave -10: ArgoCD and Linkerd CRDs (highest priority)
- Wave -9: Linkerd Control Plane
- Wave -8: Other infrastructure components

##### Source Configuration Patterns

1. **Single Source Pattern**
```yaml
source:
  repoURL: {{ $appConfig.source.repoURL | default $.Values.global.repoURL }}
  targetRevision: {{ $appConfig.source.targetRevision | default $.Values.global.targetRevision }}
```
Used for simple applications with a single source repository.

2. **Multi-Source Pattern**
```yaml
sources:
  - chart: {{ $source.chart }}
    repoURL: {{ $source.repoURL }}
    targetRevision: {{ $source.targetRevision }}
  - repoURL: {{ $.Values.global.repoURL }}
    ref: app_repo
  - repoURL: https://gitlab.ba.innovatrics.net/application-platform/argocd-infra.git
    ref: infra_repo
```
Supports complex scenarios with multiple source repositories:
- Primary chart source (e.g., Helm chart repository)
- Application repository reference
- Infrastructure repository reference

##### Value Overrides

The template supports multiple levels of value overrides:
1. Global values (`.Values.global`)
2. Component-specific values (`.Values.infrastructure.$component`)
3. Source-specific configurations
4. Helm value files

##### Namespace Handling

Special namespace handling exists for critical components:
```yaml
destination:
  {{- if eq $appName "argocd" }}
  namespace: argocd
  {{- else }}
  namespace: {{ $appConfig.namespace }}
  {{- end }}
```

##### Sync Policy Configuration

Sync policies can be defined at both global and component levels:
```yaml
syncPolicy:
  {{- if $appConfig.syncPolicy }}
  {{- toYaml $appConfig.syncPolicy | nindent 4 }}
  {{- else }}
  {{- toYaml $.Values.global.syncPolicy | nindent 4 }}
  {{- end }}
```

### Usage Examples

#### Basic Component Configuration
```yaml
infrastructure:
  argocd:
    enabled: true
    namespace: argocd
    sources:
      - repoURL: https://argoproj.github.io/argo-helm
        chart: argo-cd
        targetRevision: "7.8.2"
```

#### Multi-Source Configuration with Value Overrides
```yaml
infrastructure:
  linkerd-control-plane:
    enabled: true
    namespace: linkerd
    sources:
      - repoURL: https://helm.linkerd.io/stable
        chart: linkerd-control-plane
        targetRevision: "1.16.11"
        helm:
          valueFiles:
            - $infra_repo/infrastructure/linkerd/values.yaml
```
