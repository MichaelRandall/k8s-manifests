# Local Testing Runbook (Minikube)

This runbook is only for local development and testing on Minikube.

For production-style CI/CD and registry-based rollout, use [docs/PRODUCTION_RUNBOOK.md](docs/PRODUCTION_RUNBOOK.md).

## Scope

Use this runbook when you want to:
- Build images locally with Minikube
- Deploy quickly for local testing
- Debug pods and ingress on your machine

## Prerequisites

```bash
minikube status
kubectl version --client
```

## One-Time Cluster Setup

```bash
# Start cluster
minikube start

# Enable ingress
minikube addons enable ingress
minikube addons enable ingress-dns

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

## Full Local Test Flow

### 1. Build local images into Minikube

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube

minikube image build -t practice-backend:v5 ./backend
minikube image build -t practice-frontend:v2 ./frontend
```

### 2. Deploy local manifests

```bash
kubectl apply -f k8s-manifests/overlays/local/app-deployment.yaml \
  -f k8s-manifests/base/app-ingress.yaml
```

### 3. Restart and verify rollout

```bash
kubectl rollout restart deployment/backend-deployment
kubectl rollout restart deployment/frontend-deployment

kubectl rollout status deployment/backend-deployment --timeout=180s
kubectl rollout status deployment/frontend-deployment --timeout=180s
```

### 4. Verify resources

```bash
kubectl get pods,svc,ingress
kubectl get pods -o wide
```

### 5. Expose ingress locally

```bash
kubectl -n ingress-nginx port-forward service/ingress-nginx-controller 18080:80
```

In a second terminal:

```bash
curl http://localhost:18080/api/data
curl http://localhost:18080/ | head -n 20
xdg-open http://localhost:18080/
```

## Daily Update Loop (Local)

```bash
# 1) change code
# 2) rebuild affected image
minikube image build -t practice-backend:v5 ./backend
# or
minikube image build -t practice-frontend:v2 ./frontend

# 3) restart deployment
kubectl rollout restart deployment/backend-deployment
# or
kubectl rollout restart deployment/frontend-deployment

# 4) verify
kubectl rollout status deployment/backend-deployment --timeout=180s
kubectl rollout status deployment/frontend-deployment --timeout=180s
```

## Troubleshooting (Local)

### Image or startup issues

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Service routing issues

```bash
kubectl get svc
kubectl get endpoints backend-service
kubectl get endpoints frontend-service
```

### Ingress issues

```bash
kubectl get ingress
kubectl describe ingress practice-app-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

### In-cluster connectivity test

```bash
kubectl run curl-test --rm -i --restart=Never \
  --image=curlimages/curl:8.8.0 -- \
  curl http://backend-service:3000/api/data
```

## Cleanup (Local)

```bash
kubectl delete -f k8s-manifests/overlays/local/app-deployment.yaml \
  -f k8s-manifests/base/app-ingress.yaml

minikube stop
# optional full reset:
# minikube delete
```
