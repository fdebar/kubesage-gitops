# KubeSage GitOps

GitOps configuration and Kubernetes deployment manifests for [KubeSage](https://github.com/fdebar/kubesage).

This repository contains the Argo CD applications, Helm charts, and environment-specific configuration required to deploy and operate KubeSage on Kubernetes.

## Architecture

The KubeSage project is split across dedicated repositories:

| Repository                                             | Responsibility                                          |
| ------------------------------------------------------ | ------------------------------------------------------- |
| [kubesage](https://github.com/fdebar/kubesage)         | Backend, API and KubeSage Helm chart                    |
| [kubesage-web](https://github.com/fdebar/kubesage-web) | React frontend and Docker image                         |
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
│    kubesage-web      │
│ React frontend       │
│ Docker image         │
└──────────┬───────────┘
           │
           │ Docker images
           ▼
┌────────────────────────────────┐
│       kubesage-gitops          │
│                                │
│  Argo CD                       │
│  Helm                          │
│  Monitoring                    │
│  Environment configuration     │
└───────────────┬────────────────┘
                │
                │ GitOps
                ▼
┌────────────────────────────────┐
│          Kubernetes            │
│                                │
│  KubeSage API                  │
│  KubeSage Web                  │
│  Ingress                        │
│  Prometheus                     │
│  Loki                           │
│  Tempo                          │
│  Alloy                          │
│  OpenTelemetry Collector        │
│  Grafana                        │
└────────────────────────────────┘
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
    ├── kubesage.yaml
    └── monitoring.yaml
```

Environment configuration is kept separate from reusable Helm charts so that additional environments can be introduced without duplicating the charts.

## KubeSage Deployment

KubeSage is deployed using the Helm chart maintained in the `kubesage` repository.

The deployment includes:

```text
                    KubeSage
                       │
              ┌────────┴────────┐
              │                 │
           Backend             Web
              │                 │
          API :8000            :80
              │                 │
              └────────┬────────┘
                       │
                    Ingress
                       │
                kubesage.local
```

The Kubernetes Services for the API and Web frontend remain internal `ClusterIP` services.

External access is handled by the Kubernetes Ingress.

### Ingress routing

The KubeSage Ingress routes requests according to the path:

```text
http://kubesage.local/
        │
        └──► kubesage-web:80

http://kubesage.local/api
        │
        └──► kubesage-api:8000
```

The frontend therefore communicates with the API through the same host using the `/api` path.

This avoids exposing the API and Web frontend through separate external endpoints.

## Release and GitOps Flow

Application releases are built independently from the GitOps repository.

### Backend

A KubeSage backend release produces a Docker image and updates the KubeSage Helm chart version.

### Web

A KubeSage Web release produces a Docker image and publishes it to GitHub Container Registry.

The release image is tagged with:

```text
vX.Y.Z
```

and the corresponding Git commit SHA.

The image is also:

* scanned with Trivy
* generated with SBOM metadata
* generated with provenance metadata
* signed with Cosign
* signature-verified before deployment

### GitOps update

After a successful Web release, the release workflow automatically creates a pull request against this repository.

The development environment is updated with the new image tag and digest:

```yaml
web:
  image:
    tag: vX.Y.Z
    digest: sha256:...
```

The same GitOps process is used for KubeSage application releases.

The resulting workflow is:

```text
Application repository
        │
        │ release
        ▼
GitHub Actions
        │
        ├── Build Docker image
        ├── Security scan
        ├── SBOM / provenance
        ├── Cosign signature
        │
        ▼
GitHub Container Registry
        │
        │ GitOps PR
        ▼
kubesage-gitops
        │
        │ merge
        ▼
Argo CD
        │
        │ reconcile
        ▼
Kubernetes
```

GitOps changes are therefore reviewed before they are applied to the cluster.

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

Telemetry collection is handled through Alloy and OpenTelemetry components.

The stack provides:

* Prometheus — metrics
* Loki — logs
* Tempo — distributed traces
* Grafana — visualization and investigation
* Alloy — telemetry collection and forwarding
* OpenTelemetry Collector — telemetry processing

All components are deployed and managed through Helm and Argo CD.

## GitOps Workflow

Changes to the Kubernetes desired state are made through Git.

```text
Developer
    │
    │ pull request
    ▼
GitHub
    │
    │ merge
    ▼
Git repository
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

This keeps the Kubernetes environment:

* declarative
* version-controlled
* reproducible
* auditable

## Local Development with Minikube

KubeSage can be deployed locally using [Minikube](https://minikube.sigs.k8s.io/).

The recommended local setup uses:

* Minikube
* Docker driver
* NGINX Ingress Controller
* LoadBalancer Service
* `minikube tunnel`

### Prerequisites

* Docker
* Minikube
* `kubectl`
* Helm
* Argo CD

### Start Minikube

```bash
minikube start --driver=docker
```

Enable the NGINX Ingress Controller:

```bash
minikube addons enable ingress
```

Verify the controller:

```bash
kubectl get pods -n ingress-nginx
```

The `nginx` IngressClass should also be available:

```bash
kubectl get ingressclass
```

Expected:

```text
NAME    CONTROLLER
nginx   k8s.io/ingress-nginx
```

### Configure the Ingress Controller as a LoadBalancer

The Minikube NGINX Ingress addon normally exposes the controller as a `NodePort`.

For local access from the host, it can be changed to a `LoadBalancer`:

```bash
kubectl patch svc ingress-nginx-controller \
  -n ingress-nginx \
  -p '{"spec":{"type":"LoadBalancer"}}'
```

Then start the Minikube tunnel:

```bash
sudo minikube tunnel
```

The controller should receive a local external IP:

```bash
kubectl get svc -n ingress-nginx
```

For example:

```text
NAME                       TYPE           EXTERNAL-IP
ingress-nginx-controller   LoadBalancer   127.0.0.1
```

The tunnel process must remain running while the local cluster is being used.

### Configure local DNS

Add the KubeSage hostname to `/etc/hosts`:

```text
127.0.0.1 kubesage.local
```

Then the application is available at:

```text
http://kubesage.local
```

The resulting local architecture is:

```text
┌──────────────────────┐
│       Browser        │
│                      │
│ kubesage.local       │
└──────────┬───────────┘
           │
           │ 127.0.0.1:80
           ▼
┌──────────────────────┐
│   minikube tunnel    │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ NGINX Ingress        │
│ LoadBalancer         │
└──────────┬───────────┘
           │
       ┌───┴────┐
       │        │
       ▼        ▼
     Web       API
     :80      :8000
```

### Validate the deployment

Check the KubeSage resources:

```bash
kubectl get pods -n kubesage
kubectl get svc -n kubesage
kubectl get ingress -n kubesage
```

The Ingress should report:

```text
HOSTS            ADDRESS
kubesage.local   127.0.0.1
```

Test the frontend:

```bash
curl http://kubesage.local
```

Test the API through the Ingress:

```bash
curl http://kubesage.local/api/v1/settings
```

The same hostname is therefore used for both the frontend and API.

## Helm Validation

From the monitoring chart directory:

```bash
cd charts/monitoring
helm dependency update
helm lint .
helm template monitoring . -f ../../environments/dev/monitoring.yaml
```

The KubeSage Helm chart itself is maintained in the `kubesage` repository.

## Deploy with Argo CD

The Argo CD applications can be applied to a cluster with:

```bash
kubectl apply -f applications/kubesage.yaml
kubectl apply -f applications/monitoring.yaml
```

Argo CD then manages the lifecycle of the applications.

Check the applications with:

```bash
kubectl get applications -n argocd
```

## Design Principles

This repository follows a few core principles:

* **Git is the source of truth** for Kubernetes configuration.
* **Argo CD** continuously reconciles the desired state.
* **Helm** provides reusable and versioned Kubernetes packaging.
* **Environment-specific configuration** is kept separate from reusable charts.
* **Application code and deployment configuration** are maintained in separate repositories.
* **Application releases are promoted through GitOps pull requests.**
* **Container images are immutable and referenced by version and digest.**
* **Ingress provides the external entry point for KubeSage.**
* **Local Minikube configuration remains separate from the reusable application charts.**
* Changes to infrastructure are reviewed and version-controlled through Git.
