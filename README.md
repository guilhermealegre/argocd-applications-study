# ArgoCD Applications Study

This repository contains a GitOps setup for managing a local Kubernetes cluster using ArgoCD.

## 📁 Repository Structure

```
argocd-applications-study/
├── app-of-apps.yaml              # Root ArgoCD Application
├── apps/                          # ArgoCD Application definitions
│   ├── grafana.yaml              # Grafana Application
│   ├── prometheus.yaml           # Prometheus Application
│   ├── microservice-a.yaml       # Sample microservice Application
│   └── gateway-api.yaml          # Gateway API Application
└── manifests/                     # Kubernetes manifests and Helm values
    ├── gateway-api/
    │   └── gateway.yaml          # Main Gateway resource
    ├── grafana/
    │   ├── values.yaml           # Grafana Helm values
    │   └── httproute.yaml        # Grafana HTTPRoute
    ├── prometheus/
    │   ├── values.yaml           # Prometheus Helm values
    │   └── httproute.yaml        # Prometheus HTTPRoute
    └── microservice-a/
        ├── deployment.yaml       # Deployment manifest
        ├── service.yaml          # Service manifest
        └── httproute.yaml        # Microservice-A HTTPRoute
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

### 6. Access Your Services

You have **two options** for accessing your services:

#### Option A: LoadBalancer + Minikube Tunnel (Simple)

In a separate terminal, run:

```bash
minikube tunnel
```

This enables LoadBalancer services to get external IPs.

Access services via:
- Prometheus: `http://<EXTERNAL_IP>:9090`
- Grafana: `http://<EXTERNAL_IP>:3000`
- Microservice-A: `http://<EXTERNAL_IP>:3001`

#### Option B: Gateway API (Recommended - Production-like)

Install Gateway API for cleaner URLs:

```bash
# See GATEWAY-API-SETUP.md for detailed instructions
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.1.0/nginx-gateway.yaml
kubectl apply -f manifests/gateway-api/
```

Access services via:
- Prometheus: `http://prometheus.local`
- Grafana: `http://grafana.local`
- Microservice-A: `http://microservice-a.local`

📖 **Full Gateway API guide**: See `GATEWAY-API-SETUP.md`

## 📦 Deployed Applications

### Prometheus

- **Namespace**: `monitoring`
- **Service Type**: LoadBalancer
- **Port**: 9090
- **Features**:
  - Kubernetes metrics collection (nodes, pods, services)
  - cAdvisor metrics (container metrics)
  - Kube-state-metrics (cluster state)
  - Node exporter (system metrics)
- **Access**: `http://<EXTERNAL_IP>:9090`

### Grafana

- **Namespace**: `monitoring`
- **Service Type**: LoadBalancer
- **Port**: 3000
- **Default Credentials**:
  - Username: `admin`
  - Password: `admin`
- **Pre-configured**: Prometheus datasource already connected
- **Access**: `http://<EXTERNAL_IP>:3000`

### Microservice A

- **Namespace**: `serviceA`
- **Service Type**: LoadBalancer
- **Image**: `nginx:latest`
- **Port**: 3001
- **Prometheus**: Annotations added for metric scraping
- **Access**: `http://<EXTERNAL_IP>:3001`

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

### Get Service Access URLs

```bash
# Get the shared external IP (minikube tunnel uses same IP for all services)
EXTERNAL_IP=$(kubectl get svc -n monitoring prometheus-server -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Display all service URLs with their unique ports
echo "🔍 Prometheus: http://$EXTERNAL_IP:9090"
echo "📊 Grafana:    http://$EXTERNAL_IP:3000"
echo "🚀 Microservice-A: http://$EXTERNAL_IP:3001"
```

Or check individual services:

```bash
kubectl get svc -n monitoring
kubectl get svc -n serviceA
```

### View All Pods

```bash
kubectl get pods --all-namespaces
```

## 🔧 Configuration

### Prometheus

The Prometheus Helm chart is configured with:
- **Service type**: LoadBalancer on port 9090
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
- **Service type**: LoadBalancer on port 3000
- **Persistence**: Disabled (for simplicity)
- **Datasource**: Pre-configured Prometheus connection at `http://prometheus-server.monitoring.svc.cluster.local:9090`
- **Auto-refresh**: 30s interval for dashboards

### Microservice A

A simple nginx-based microservice with:
- 1 replica
- Resource limits set (100m-200m CPU, 128-256Mi RAM)
- LoadBalancer service on port 3001 (container runs on 80)
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
- **Two access options**:
  - **LoadBalancer**: Requires `minikube tunnel` to be running
  - **Gateway API**: More production-like with clean URLs (see `GATEWAY-API-SETUP.md`)
- All sync policies are set to automated for convenience
- Prometheus and Grafana deploy to `monitoring` namespace
- Microservice A deploys to `serviceA` namespace

## 🌐 Gateway API Alternative

For a more production-like setup with clean URLs, see **[GATEWAY-API-SETUP.md](GATEWAY-API-SETUP.md)**

Benefits:
- ✅ Single entry point
- ✅ Clean URLs: `http://grafana.local` instead of `http://192.168.49.2:3000`
- ✅ No need for minikube tunnel
- ✅ Production-ready routing

