# KubeSage GitOps

GitOps configuration and Kubernetes deployment manifests for [KubeSage](https://github.com/fdebar/kubesage).

This repository contains the Argo CD applications, Helm charts, and environment-specific configuration required to deploy and operate KubeSage on Kubernetes.

## Architecture

The KubeSage project is split across dedicated repositories:

| Repository                                             | Responsibility                                          |
| ------------------------------------------------------ | ------------------------------------------------------- |
| [kubesage](https://github.com/fdebar/kubesage)         | Backend, API and KubeSage Helm chart                    |
| [kubesage-web](https://github.com/fdebar/kubesage-web) | React frontend                                          |
| **kubesage-gitops**                                    | Argo CD, monitoring stack and environment configuration |

The deployment flow is:

```text
┌──────────────────────┐
│      kubesage        │
│ Backend / API        │
│ Helm chart           │
└──────────┬───────────┘
           │
           │ Docker image
           │ Helm chart
           ▼
┌──────────────────────┐
│   kubesage-gitops    │
│                      │
│  Argo CD             │
│  Helm                │
│  Monitoring          │
│  Environment config  │
└──────────┬───────────┘
           │
           │ GitOps
           ▼
┌──────────────────────┐
│     Kubernetes       │
│                      │
│ KubeSage             │
│ Prometheus           │
│ Loki                 │
│ Tempo                │
│ Alloy                │
│ Grafana              │
└──────────────────────┘
```

## Repository Structure

```text
kubesage-gitops/
├── applications/
│   ├── kubesage.yaml
│   └── monitoring.yaml
│
├── charts/
│   └── monitoring/
│       ├── Chart.yaml
│       ├── dashboard/
│       └── templates/
│
└── environments/
    └── dev/
        ├── kubesage.yaml
        └── monitoring.yaml
```

### `applications/`

Contains the [Argo CD](https://argo-cd.readthedocs.io/) `Application` resources used to deploy KubeSage and its monitoring stack.

* `kubesage.yaml` — KubeSage application
* `monitoring.yaml` — observability stack

Argo CD continuously reconciles these applications with the desired state defined in Git.

### `charts/monitoring/`

Contains the Helm chart used to deploy the KubeSage observability stack.

The chart currently includes:

* Prometheus
* Grafana
* Loki
* Tempo
* Alloy
* OpenTelemetry Collector

It also contains KubeSage-specific Grafana dashboards and Kubernetes resources.

### `environments/`

Contains environment-specific configuration.

Currently:

```text
environments/
└── dev/
```

Environment configuration is kept separate from the reusable Helm chart so that additional environments can be introduced without duplicating the chart itself.

## Observability Stack

KubeSage uses an integrated observability stack:

```text
                    ┌──────────────┐
                    │   KubeSage   │
                    └──────┬───────┘
                           │
                ┌──────────┼──────────┐
                │          │          │
              Metrics     Logs      Traces
                │          │          │
                ▼          ▼          ▼
           Prometheus     Loki      Tempo
                │          │          │
                └──────────┼──────────┘
                           ▼
                        Grafana
```

All components are deployed and managed through Helm and Argo CD.

## GitOps Workflow

Changes to the Kubernetes desired state are made through Git.

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    │ reconciliation
    ▼
Argo CD
    │
    │ apply desired state
    ▼
Kubernetes
```

Argo CD is responsible for continuously reconciling the cluster with the configuration stored in this repository.

This keeps the Kubernetes environment declarative, version-controlled, and reproducible.

## Local Development

### Prerequisites

* Kubernetes cluster
* `kubectl`
* Helm
* Argo CD

### Validate the Helm chart

From the monitoring chart directory:

```bash
cd charts/monitoring
helm dependency update
helm lint .
helm template monitoring . -f ../../environments/dev/monitoring.yaml
```

### Deploy with Argo CD

The Argo CD applications can be applied to a cluster with:

```bash
kubectl apply -f applications/kubesage.yaml
kubectl apply -f applications/monitoring.yaml
```

Argo CD then manages the lifecycle of the applications.

## Design Principles

This repository follows a few core principles:

* **Git is the source of truth** for Kubernetes configuration.
* **Argo CD** continuously reconciles the desired state.
* **Helm** provides reusable and versioned Kubernetes packaging.
* **Environment-specific configuration** is kept separate from reusable charts.
* **Application code and deployment configuration** are maintained in separate repositories.
* Changes to infrastructure are reviewed and version-controlled through Git.
