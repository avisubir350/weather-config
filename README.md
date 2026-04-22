# Weather Dashboard - ArgoCD GitOps Setup

A live weather dashboard application deployed using ArgoCD GitOps methodology on Kubernetes. The application provides real-time weather information and news feeds with a modern glass-morphism UI.

## Features

- 🌤️ Real-time weather data with 6-hour forecasts
- 📰 Live news feeds across multiple categories
- 🎨 Modern glass-morphism UI design
- 🔄 Auto-refresh every 5 minutes
- 📱 Responsive design for all devices

## Architecture

- **Frontend**: Flask web application with modern CSS
- **Backend**: Python Flask API
- **Deployment**: Kubernetes with ArgoCD GitOps
- **Ingress**: NGINX Ingress Controller
- **Container**: Docker image `avisubir/weather-news-dashboard:7fd2645`

## Prerequisites

- Kubernetes cluster (Kind/Minikube/EKS/GKE/AKS)
- kubectl configured
- Git

## Setup Instructions

### 1. Create Kubernetes Cluster (Kind Example)

```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name argocd-cluster
```

### 2. Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
```

### 3. Access ArgoCD UI

- URL: https://localhost:8080
- Username: `admin`
- Password: (from step 2 above)

### 4. Install NGINX Ingress Controller

```bash
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/kind/deploy.yaml

# Label node for ingress (Kind specific)
kubectl label node argocd-cluster-control-plane ingress-ready=true

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=90s
```

### 5. Create Weather Namespace

```bash
kubectl create namespace weather
```

### 6. Deploy Application via ArgoCD

#### Option A: Using ArgoCD CLI

```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login to ArgoCD
argocd login localhost:8080 --username admin --password <password> --insecure

# Create application
argocd app create weather \
  --repo https://github.com/avisubir350/weather-config.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace weather \
  --sync-policy automated \
  --auto-prune

# Sync application
argocd app sync weather
```

#### Option B: Using ArgoCD UI

1. Go to https://localhost:8080
2. Click "NEW APP"
3. Fill in:
   - Application Name: `weather`
   - Project: `default`
   - Repository URL: `https://github.com/avisubir350/weather-config.git`
   - Path: `.`
   - Cluster URL: `https://kubernetes.default.svc`
   - Namespace: `weather`
4. Enable "AUTO-SYNC" and "PRUNE RESOURCES"
5. Click "CREATE"

### 7. Access the Application

#### Method 1: Port Forward (Recommended)

```bash
# Port forward the service
kubectl port-forward -n weather svc/weather-news-service 8081:80

# Access in browser
open http://localhost:8081
```

#### Method 2: Ingress Controller

```bash
# Port forward ingress controller
kubectl port-forward --namespace=ingress-nginx service/ingress-nginx-controller 8081:80

# Access in browser with Host header
curl -H "Host: localhost" http://localhost:8081
# Or open http://localhost:8081 in browser
```

## Repository Structure

```
weather-config/
├── README.md              # This file
├── deployment.yml         # Kubernetes Deployment
├── service.yml           # Kubernetes Service
├── ingress.yml           # Kubernetes Ingress
└── project.yml           # ArgoCD Application manifest
```

## Configuration Files

### deployment.yml
- Deploys 2 replicas of the weather dashboard
- Uses image: `avisubir/weather-news-dashboard:7fd2645`
- Exposes port 5000 (Flask default)

### service.yml
- ClusterIP service
- Routes traffic from port 80 to container port 5000

### ingress.yml
- NGINX ingress configuration
- Routes traffic from `localhost` to the service

### project.yml
- ArgoCD Application definition
- Enables GitOps workflow with auto-sync

## Troubleshooting

### Application Not Accessible

1. Check pod status:
```bash
kubectl get pods -n weather
```

2. Check service endpoints:
```bash
kubectl get endpoints -n weather
```

3. Check ingress status:
```bash
kubectl get ingress -n weather
```

4. View pod logs:
```bash
kubectl logs -n weather deployment/weather-news-deployment
```

### ArgoCD Sync Issues

1. Check application status:
```bash
argocd app get weather
```

2. Manual sync:
```bash
argocd app sync weather
```

3. View sync history:
```bash
argocd app history weather
```

## Development

To modify the application:

1. Update the configuration files
2. Commit and push changes
3. ArgoCD will automatically detect and sync changes (if auto-sync enabled)

## Clean Up

```bash
# Delete application
argocd app delete weather

# Delete namespace
kubectl delete namespace weather

# Delete ArgoCD
kubectl delete namespace argocd

# Delete ingress controller
kubectl delete namespace ingress-nginx

# Delete Kind cluster
kind delete cluster --name argocd-cluster
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test the deployment
5. Submit a pull request

## License

This project is open source and available under the MIT License.
