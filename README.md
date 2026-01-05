# Universal Microservice Helm Chart

![Version: 1.0.0](https://img.shields.io/badge/Version-1.0.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg?style=flat-square)

A **cloud-agnostic**, production-grade Library Chart designed to standardize microservice deployments across **On-Premise (Talos Linux)**, **Proxmox**, and **Cloud** environments.

This chart removes vendor lock-in (replacing GKE-specific annotations and proxies with standard Kubernetes primitives) and is optimized for **GitOps workflows** with ArgoCD.

## 🚀 Key Features

* **Cloud Agnostic:** Zero dependency on cloud-specific CLIs or proprietary annotations. Works seamlessly on bare-metal Talos clusters.
* **Smart Defaults:** Automatically configures Ingress paths, Service ports, and Health Checks based on a single `contextPath` and `containerPort`.
* **GitOps Ready:** Designed for "Configuration as Code." Override environment-specific logic (Dev vs. Prod) purely through `values.yaml`.
* **Observability First:** Built-in `ServiceMonitor` integration for the Prometheus Operator.
* **Secure by Design:** Supports `envFrom` for bulk loading **SealedSecrets** and enforces non-root security contexts by default.
* **Dynamic Image Logic:** intelligently constructs image URLs using `registry`, `project`, and `applicationName` to standardize your internal Harbor usage.

## 📋 Prerequisites

* Kubernetes 1.19+
* Helm 3.0+
* (Optional) Prometheus Operator (for `ServiceMonitor`)
* (Optional) Cert-Manager (for automatic TLS)

## 📦 Installation

To add the repository and install the chart:

```console
helm repo add universal-chart [https://tatahostit.github.io/universal-chart-repo](https://tatahostit.github.io/universal-chart-repo)
helm install my-app universal-chart/universal-app

```

*(Note: Replace the URL above with your actual GitHub Pages URL once published)*

## ⚙️ Configuration

The chart uses a "Smart Configuration" approach. Instead of editing complex templates, you define high-level attributes, and the chart handles the wiring.

| Parameter | Description | Default |
| --- | --- | --- |
| **Global Identity** |  |  |
| `applicationName` | Used for Deployment name, Service name, and default Image name. | `my-app` |
| `environment` | Injected as `APP_ENV` variable. | `dv` |
| `namespace` | (Optional) Explicit namespace override. | `default` |
| **Smart Settings** |  |  |
| `containerPort` | The internal port your app listens on. Updates Service & Deployment. | `8080` |
| `contextPath` | Base path (e.g., `/api`). Automatically configures Ingress & Probes. | `/` |
| **Image** |  |  |
| `image.registry` | Registry URL (e.g., Harbor). | `harbor.tatahostit.com` |
| `image.project` | Project name (e.g., library). | `library` |
| `image.name` | Override image name (defaults to `applicationName`). | `""` |
| `image.tag` | Image tag. | `""` |
| **Networking** |  |  |
| `service.port` | The External Service port (ClusterIP). | `80` |
| `ingress.enabled` | Enable Ingress resource. | `false` |
| `ingress.hosts` | List of hosts. Auto-uses `contextPath` if `paths` is empty. | `[]` |
| **Observability** |  |  |
| `serviceMonitor.enabled` | Create Prometheus ServiceMonitor. | `false` |
| `livenessProbe.enabled` | Enable/Disable Liveness Probe. | `true` |
| `readinessProbe.enabled` | Enable/Disable Readiness Probe. | `true` |
| **Scaling** |  |  |
| `autoscaling.enabled` | Enable HPA (CPU/Memory). | `false` |
| `resources` | CPU/Memory Requests and Limits. | `{}` |

## 💡 How "Smart Defaults" Work

Instead of manually wiring ports and paths in 5 different places, you set them once:

```yaml
containerPort: 9000
contextPath: "/payment-service"

```

The chart automatically:

1. Configures the **Service** to target port `9000`.
2. Sets the **Ingress** rule to route `/payment-service` to your app.
3. Configures **Health Checks** to probe `/payment-service/health`.

## 🛠 Usage Examples

### 1. Development (Minimal & Fast)

*Single replica, no monitoring, disabled probes for easy debugging.*

```yaml
applicationName: "user-service"
environment: "dv"
containerPort: 8080
contextPath: "/users"

image:
  tag: "dev-latest"
  pullPolicy: Always

# Disable probes to allow debugging of broken apps
livenessProbe:
  enabled: false
readinessProbe:
  enabled: false

# Minimal resources
resources:
  requests:
    cpu: "50m"
    memory: "128Mi"

```

### 2. Production (High Availability)

*Autoscaling, strict probes, monitoring enabled, and pinned versions.*

```yaml
applicationName: "user-service"
environment: "pr"
containerPort: 8080
contextPath: "/users"

image:
  tag: "v1.2.0" # Always pin versions in prod

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 8

ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.tatahostit.com

# Enable Prometheus scraping
serviceMonitor:
  enabled: true
  labels:
    release: prometheus

```

### 3. ArgoCD Integration

This chart is designed to be consumed as a **Helm Source** in ArgoCD.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-service
spec:
  source:
    repoURL: '[https://github.com/TataHostIT/universal-chart-repo.git](https://github.com/TataHostIT/universal-chart-repo.git)'
    targetRevision: 1.0.0
    path: charts/universal-app
    helm:
      valueFiles:
        - values-pr.yaml

```
