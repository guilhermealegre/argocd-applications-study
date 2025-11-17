# Gateway API Setup Guide

This guide explains how to use **Gateway API** instead of `minikube tunnel` + LoadBalancer services.

## 🎯 Benefits of Gateway API

✅ **Single Entry Point** - One Gateway for all services  
✅ **Host-based Routing** - Access via domain names (e.g., `grafana.local`)  
✅ **Production-like** - Similar to cloud Load Balancers  
✅ **No minikube tunnel** - Works with standard minikube  
✅ **Clean URLs** - No need to remember ports

---

## 🚀 Installation Steps

### Step 1: Install Gateway API CRDs

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

### Step 2: Install NGINX Gateway Controller

```bash
# Install NGINX Gateway Fabric (Gateway API implementation)
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.1.0/crds.yaml
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.1.0/nginx-gateway.yaml
```

Wait for the Gateway controller to be ready:

```bash
kubectl wait --namespace nginx-gateway --for=condition=available --timeout=300s deployment/nginx-gateway
```

### Step 3: Deploy Gateway API Resources via ArgoCD

The Gateway and HTTPRoutes are already included in your repository:

- **Gateway**: Deployed via `apps/gateway-api.yaml` → `manifests/gateway-api/gateway.yaml`
- **HTTPRoutes**: Each application deploys its own HTTPRoute:
  - Prometheus: `manifests/prometheus/httproute.yaml`
  - Grafana: `manifests/grafana/httproute.yaml`
  - Microservice-A: `manifests/microservice-a/httproute.yaml`

They will be deployed automatically by ArgoCD when you sync the applications.

Or manually apply:

```bash
# Deploy Gateway first
kubectl apply -f manifests/gateway-api/gateway.yaml

# Then deploy HTTPRoutes (automatically deployed with each app)
kubectl apply -f manifests/prometheus/httproute.yaml
kubectl apply -f manifests/grafana/httproute.yaml
kubectl apply -f manifests/microservice-a/httproute.yaml
```

### Step 4: Get Gateway External IP

```bash
# Get the Gateway LoadBalancer IP
kubectl get gateway main-gateway -n monitoring

# Get the external IP
GATEWAY_IP=$(kubectl get svc -n nginx-gateway nginx-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Gateway IP: $GATEWAY_IP"
```

### Step 5: Configure Local DNS

Add entries to your `/etc/hosts` file:

```bash
# Get the Gateway IP
GATEWAY_IP=$(kubectl get svc -n nginx-gateway nginx-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Add to /etc/hosts (requires sudo)
echo "$GATEWAY_IP prometheus.local" | sudo tee -a /etc/hosts
echo "$GATEWAY_IP grafana.local" | sudo tee -a /etc/hosts
echo "$GATEWAY_IP microservice-a.local" | sudo tee -a /etc/hosts
```

Or do it manually:

```bash
sudo nano /etc/hosts
```

Add these lines (replace with your actual Gateway IP):
```
192.168.49.2  prometheus.local
192.168.49.2  grafana.local
192.168.49.2  microservice-a.local
```

---

## 🌐 Access Your Services

After setup, access services via clean URLs:

| Service        | URL                              | Credentials        |
|---------------|----------------------------------|-------------------|
| Prometheus    | http://prometheus.local          | N/A               |
| Grafana       | http://grafana.local             | admin / admin     |
| Microservice-A| http://microservice-a.local      | N/A               |

### Quick Test

```bash
# Test Prometheus
curl -s http://prometheus.local | grep "<title>"

# Test Grafana
curl -s http://grafana.local | grep "<title>"

# Test Microservice-A
curl -s http://microservice-a.local | grep "nginx"
```

---

## 🔍 Verify Gateway Setup

### Check Gateway Status

```bash
kubectl get gateway -n monitoring
```

Expected output:
```
NAME           CLASS   ADDRESS        PROGRAMMED   AGE
main-gateway   nginx   192.168.49.2   True         5m
```

### Check HTTPRoutes

```bash
kubectl get httproute -n monitoring
```

Expected output:
```
NAME                   HOSTNAMES                 AGE
prometheus-route       ["prometheus.local"]      5m
grafana-route         ["grafana.local"]         5m
microservice-a-route  ["microservice-a.local"]  5m
```

### Check Gateway Controller

```bash
kubectl get pods -n nginx-gateway
kubectl logs -n nginx-gateway deployment/nginx-gateway
```

---

## 🔄 Architecture Comparison

### Before (LoadBalancer)
```
Internet/localhost
    |
    ├─> LoadBalancer (192.168.49.2:9090) -> Prometheus
    ├─> LoadBalancer (192.168.49.2:3000) -> Grafana
    └─> LoadBalancer (192.168.49.2:8080) -> Microservice-A
```

### After (Gateway API)
```
Internet/localhost
    |
    └─> Gateway (192.168.49.2:80)
         ├─> prometheus.local -> Prometheus Service
         ├─> grafana.local -> Grafana Service
         └─> microservice-a.local -> Microservice-A Service
```

---

## 🎯 Advanced Routing Examples

### Path-based Routing

You can also route based on paths instead of hostnames:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: services-route
  namespace: monitoring
spec:
  parentRefs:
    - name: main-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /prometheus
      backendRefs:
        - name: prometheus-server
          port: 9090
    
    - matches:
        - path:
            type: PathPrefix
            value: /grafana
      backendRefs:
        - name: grafana
          port: 3000
```

Access via:
- http://192.168.49.2/prometheus
- http://192.168.49.2/grafana

### TLS/HTTPS Support

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
spec:
  gatewayClassName: nginx
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-tls-cert
```

---

## 🧹 Cleanup Gateway API

### Remove Gateway Resources

```bash
kubectl delete -f manifests/gateway-api/
```

### Uninstall NGINX Gateway

```bash
kubectl delete -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.1.0/nginx-gateway.yaml
kubectl delete -f https://github.com/nginxinc/nginx-gateway-fabric/releases/download/v1.1.0/crds.yaml
```

### Remove Gateway API CRDs

```bash
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml
```

### Clean /etc/hosts

```bash
sudo nano /etc/hosts
# Remove the lines with .local domains
```

---

## 🆚 Should You Use Gateway API or LoadBalancer?

| Feature              | LoadBalancer + Tunnel | Gateway API           |
|---------------------|----------------------|----------------------|
| Setup Complexity    | Simple               | Moderate             |
| Production Ready    | ❌                    | ✅                   |
| Clean URLs          | ❌                    | ✅                   |
| Single Entry Point  | ❌                    | ✅                   |
| SSL/TLS Support     | Limited              | ✅ Built-in          |
| Path Routing        | ❌                    | ✅                   |
| Cloud Compatible    | ✅                    | ✅                   |

**Recommendation**: Use **Gateway API** for a more production-like setup!

---

## 🐛 Troubleshooting

### Gateway Not Getting an IP

```bash
# Check Gateway controller is running
kubectl get pods -n nginx-gateway

# Check service
kubectl get svc -n nginx-gateway
```

### HTTPRoute Not Working

```bash
# Check HTTPRoute status
kubectl describe httproute grafana-route -n monitoring

# Check Gateway controller logs
kubectl logs -n nginx-gateway -l app.kubernetes.io/name=nginx-gateway
```

### Domain Not Resolving

```bash
# Verify /etc/hosts
cat /etc/hosts | grep ".local"

# Test DNS resolution
ping prometheus.local
```

### Service Not Found

```bash
# Check if services exist
kubectl get svc -n monitoring
kubectl get svc -n serviceA

# Check if services have correct ports
kubectl describe svc grafana -n monitoring
```

---

## 📚 Resources

- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
- [NGINX Gateway Fabric](https://docs.nginx.com/nginx-gateway-fabric/)
- [Gateway API Examples](https://gateway-api.sigs.k8s.io/guides/)

---

## 🎉 Summary

With Gateway API, you get:
- **Clean URLs**: `http://grafana.local` instead of `http://192.168.49.2:3000`
- **Single Entry Point**: One Gateway for all services
- **Production-like**: Matches how cloud providers work
- **Better Control**: Advanced routing, TLS, traffic splitting, etc.

Enjoy your modern Kubernetes routing! 🚀

