# ArgoCD Applications Study

This repository contains a GitOps setup for managing a local Kubernetes cluster using ArgoCD.

## 📁 Repository Structure

```
argocd-applications-study/
├── app-of-apps.yaml              # Root ArgoCD Application
├── apps/                          # ArgoCD Application definitions
│   ├── grafana.yaml              # Grafana Application
│   ├── prometheus.yaml           # Prometheus Application
│   └── microservice-a.yaml       # Sample microservice Application
└── manifests/                     # Kubernetes manifests and Helm values
    ├── grafana/
    │   └── values.yaml           # Grafana Helm values
    ├── prometheus/
    │   └── values.yaml           # Prometheus Helm values
    └── microservice-a/
        ├── deployment.yaml       # Deployment manifest
        └── service.yaml          # Service manifest
```

## 🚀 Getting Started

### Prerequisites

- Minikube installed and running
- ArgoCD installed in your cluster
- kubectl configured to access your cluster

### 1. Start Minikube

```bash
minikube start --kubernetes-version=v1.30.0
```

### 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3. Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Login at: https://localhost:8080
- Username: `admin`
- Password: (from the command above)

### 4. Configure Repository URL

The repository URLs have been pre-configured to:
```
https://github.com/guilhermealegre/argocd-applications-study.git
```

If you need to change it, update the `repoURL` in:
- `app-of-apps.yaml`
- `apps/grafana.yaml` (in the sources section)
- `apps/prometheus.yaml` (in the sources section)
- `apps/microservice-a.yaml`

### 5. Deploy the App-of-Apps

```bash
kubectl apply -f app-of-apps.yaml
```

### 6. Enable LoadBalancer Services

In a separate terminal, run:

```bash
minikube tunnel
```

This enables LoadBalancer services to get external IPs.

## 📦 Deployed Applications

### Prometheus

- **Namespace**: `monitoring`
- **Service Type**: LoadBalancer
- **Port**: 80 (mapped to internal 9090)
- **Features**:
  - Kubernetes metrics collection (nodes, pods, services)
  - cAdvisor metrics (container metrics)
  - Kube-state-metrics (cluster state)
  - Node exporter (system metrics)
- **Access**: Get the external IP with `kubectl get svc -n monitoring prometheus-server`

### Grafana

- **Namespace**: `monitoring`
- **Service Type**: LoadBalancer
- **Port**: 80 (mapped to internal 3000)
- **Default Credentials**:
  - Username: `admin`
  - Password: `admin`
- **Pre-configured**: Prometheus datasource already connected
- **Access**: Get the external IP with `kubectl get svc -n monitoring grafana`

### Microservice A

- **Namespace**: `serviceA`
- **Service Type**: LoadBalancer
- **Image**: `nginx:latest`
- **Port**: 80
- **Prometheus**: Annotations added for metric scraping
- **Access**: Get the external IP with `kubectl get svc -n serviceA microservice-a`

## 🔍 Useful Commands

### Check ArgoCD Applications

```bash
kubectl get applications -n argocd
```

### Check Application Status

```bash
kubectl get application app-of-apps -n argocd -o yaml
```

### View Monitoring Services

```bash
kubectl get svc -n monitoring
```

### View Prometheus Metrics

```bash
# Get Prometheus external IP
PROMETHEUS_IP=$(kubectl get svc -n monitoring prometheus-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Prometheus: http://$PROMETHEUS_IP"
```

### View Grafana Dashboard

```bash
# Get Grafana external IP
GRAFANA_IP=$(kubectl get svc -n monitoring grafana -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Grafana: http://$GRAFANA_IP"
```

### View Microservice A Service

```bash
kubectl get svc -n serviceA microservice-a
```

### View All Pods

```bash
kubectl get pods --all-namespaces
```

## 🔧 Configuration

### Prometheus

The Prometheus Helm chart is configured with:
- **Service type**: LoadBalancer on port 80
- **Persistence**: Disabled (for simplicity)
- **Kubernetes scraping**: Configured for:
  - API servers
  - Nodes (kubelet metrics)
  - Pods (with prometheus.io/scrape annotations)
  - cAdvisor (container metrics)
  - Service endpoints
- **Components enabled**:
  - kube-state-metrics: Cluster state metrics
  - node-exporter: Node/system metrics
- **Resource limits**: 500m CPU / 512Mi RAM (configurable)

### Grafana

The Grafana Helm chart is configured with:
- **Admin credentials**: `admin/admin`
- **Service type**: LoadBalancer on port 80
- **Persistence**: Disabled (for simplicity)
- **Datasource**: Pre-configured Prometheus connection at `http://prometheus-server.monitoring.svc.cluster.local:80`
- **Auto-refresh**: 30s interval for dashboards

### Microservice A

A simple nginx-based microservice with:
- 1 replica
- Resource limits set (100m-200m CPU, 128-256Mi RAM)
- LoadBalancer service on port 80
- Prometheus scraping annotations enabled

## 🔄 Sync Policies

All applications are configured with automated sync policies:
- `prune: true` - Removes resources when deleted from Git
- `selfHeal: true` - Automatically syncs when drift is detected
- `CreateNamespace: true` - Automatically creates namespaces if they don't exist

## 🧹 Cleanup

To remove all applications:

```bash
kubectl delete -f app-of-apps.yaml
```

To completely remove ArgoCD:

```bash
kubectl delete namespace argocd
kubectl delete namespace monitoring
kubectl delete namespace serviceA
```

## 📝 Notes

- This setup is designed for local development with Minikube
- LoadBalancer services require `minikube tunnel` to be running
- All sync policies are set to automated for convenience
- Prometheus and Grafana deploy to `monitoring` namespace
- Microservice A deploys to `serviceA` namespace

