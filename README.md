# helm-chart

A generic Helm chart for deploying containerized applications to AWS EKS, with full CI/CD via GitHub Actions.

## What it does

The chart takes a list of `applications` in `values.yaml` and dynamically generates all the Kubernetes resources needed to run them on EKS — deployments, services, ingress, autoscaling, network policies, and secrets. No need to write boilerplate Kubernetes YAML per app; just define what changes between apps.

## Stack

- **Helm** — templating and packaging
- **AWS EKS** — target cluster
- **AWS ALB (Ingress Controller)** — load balancing
- **AWS Secrets Manager + External Secrets Operator** — secret injection
- **IRSA** — IAM roles for service accounts (pod-level AWS auth)
- **HPA + Metrics Server** — CPU-based autoscaling
- **GitHub Actions** — CI linting + automated chart release to GitHub Pages

## Project structure

```
charts/
└── generic-app/
    ├── Chart.yaml
    ├── values.yaml           # defaults and required fields
    ├── values.schema.json    # validates values at helm install/upgrade
    └── templates/
        ├── deployments.yaml
        ├── services.yaml
        ├── ingress.yaml
        ├── configmap.yaml
        ├── secrets.yaml          # ExternalSecret resources
        ├── service-accounts.yaml # IRSA-enabled service accounts
        ├── hpa.yaml
        ├── namespace.yaml
        ├── network-polices.yaml
        ├── resources.yaml        # shared resource helper
        └── generic-labels.yaml   # shared label/annotation helpers
.github/workflows/
    ├── ci.yaml               # lint + kubeconform on PRs
    ├── chart-releaser.yaml   # package and publish to gh-pages on merge
    └── pr-title-validation.yaml
```

## Usage

### Install from GitHub Pages

```bash
helm repo add my-charts https://<YOUR_GITHUB_USERNAME>.github.io/helm-eks/
helm repo update
helm install my-release my-charts/generic-app -f my-values.yaml
```

### Example values

```yaml
projectName: my-app
environment: production

aws:
  accountId: "123456789012"
  region: "us-east-1"

ingress:
  certificateArns:
    - "arn:aws:acm:us-east-1:123456789012:certificate/<id>"

applications:
  - type: client
    name: main
    imageRepo: "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app/client"
    imageTag: "v1.2.0"
    replicas: 2
    ingress:
      enabled: true
      host: "my-app.example.com"
      path: "/"

  - type: server
    name: main
    imageRepo: "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app/server"
    imageTag: "v1.2.0"
    replicas: 2
    probePath: "/health"
    awsRole: "arn:aws:iam::123456789012:role/my-app-server-role"
    secrets:
      - name: "db-credentials"
        path: "my-app/production/database"
```

### What gets created per application

| Resource       | Name                                           |
| -------------- | ---------------------------------------------- |
| Deployment     | `{type}-{name}`                                |
| Service        | `{type}-{name}-service`                        |
| ConfigMap      | `{type}-{name}-cm`                             |
| HPA            | `{type}-{name}-hpa`                            |
| Ingress        | `{type}-{name}-ingress` _(if enabled)_         |
| ExternalSecret | `{secret-name}-external-secret` _(per secret)_ |

All resources land in the namespace `{environment}-{projectName}` (created by the chart).

## Security defaults

Every deployment gets these out of the box:

- Runs as non-root (UID/GID 1001)
- Read-only root filesystem
- No privilege escalation
- All Linux capabilities dropped
- Network policies that deny cross-namespace ingress by default

## IRSA

When `awsRole` is set on an app, a dedicated service account is created with the annotation:

```
eks.amazonaws.com/role-arn: arn:aws:iam::{accountId}:role/{roleName}
```

Kubernetes injects a signed OIDC token into the pod, which AWS uses to assume the IAM role — no static credentials needed.

## Secrets

Secrets are synced from AWS Secrets Manager via [External Secrets Operator](https://external-secrets.io/):

```yaml
secrets:
  - name: "db-credentials" # name of the resulting k8s Secret
    path: "my-app/prod/database" # key path in AWS Secrets Manager
    keys: [DB_PASSWORD] # optional: pull only specific keys
```

Secrets refresh every 90 seconds and are mounted into pods via `envFrom`.

## CI/CD

**On pull request** (`ci.yaml`):

- yamllint on all YAML files
- `helm lint` on all charts
- `kubeconform` validates rendered manifests against the Kubernetes API schema

**On merge to main** (`chart-releaser.yaml`):

- Packages the chart with `helm package`
- Publishes it to the `gh-pages` branch as a Helm repository
- Chart is immediately installable via `helm repo add`

## Setup

1. Fork / clone this repo
2. Fill in your AWS values in `values.yaml`
3. Enable GitHub Pages in repo settings → source: `gh-pages` branch
4. Bump `version` in `Chart.yaml` and push to main — the release workflow does the rest
