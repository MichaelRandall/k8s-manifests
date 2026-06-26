# Production Runbook (CI/CD + Docker Hub + Kubernetes)

This runbook is only for production-style deployment flow:
- service repos build in GitHub Actions
- images are pushed to Docker Hub
- Kubernetes pulls from Docker Hub tags

For local Minikube image-build workflow, use [docs/RUNBOOK.md](docs/RUNBOOK.md).

## Scope

Use this runbook when you want to:
- deploy images built by CI/CD
- promote reproducible container artifacts
- roll out from a registry-backed source of truth

## Repositories

- Backend repo: https://github.com/MichaelRandall/k8s-back
- Frontend repo: https://github.com/MichaelRandall/k8s-front
- Manifests repo: https://github.com/MichaelRandall/k8s-manifests
- Docker Hub repo: https://hub.docker.com/r/mran66/k8s-test

## Image Convention

Single Docker Hub repository with tag prefixes:
- Backend: mran66/k8s-test:backend-latest
- Frontend: mran66/k8s-test:frontend-latest

## One-Time Setup

### 1. GitHub Actions secrets

Add DOCKER_HUB_TOKEN to both service repos:
- https://github.com/MichaelRandall/k8s-back/settings/secrets/actions
- https://github.com/MichaelRandall/k8s-front/settings/secrets/actions

### 2. Cluster readiness

```bash
kubectl cluster-info
kubectl get nodes
kubectl get ns
```

## Full Production Deployment Flow

### 1. Make and push backend or frontend code

Backend example:

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube/backend
git add .
git commit -m "Backend change"
git push origin main
```

Frontend example:

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube/frontend
git add .
git commit -m "Frontend change"
git push origin main
```

### 2. Verify CI build-and-push is green

- Backend Actions: https://github.com/MichaelRandall/k8s-back/actions
- Frontend Actions: https://github.com/MichaelRandall/k8s-front/actions

Confirm tags exist:
- https://hub.docker.com/r/mran66/k8s-test/tags

### 3. Apply manifests from manifests repo workspace

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube

kubectl apply -f k8s-manifests/base/app-deployment-polyrepo.yaml \
  -f k8s-manifests/base/app-ingress.yaml
```

### 4. Roll out latest tags

```bash
kubectl rollout restart deployment/backend-deployment
kubectl rollout restart deployment/frontend-deployment

kubectl rollout status deployment/backend-deployment --timeout=180s
kubectl rollout status deployment/frontend-deployment --timeout=180s
```

### 5. Verify deployed image references

```bash
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
# Expected includes:
# mran66/k8s-test:backend-latest
# mran66/k8s-test:frontend-latest
```

### 6. Validate runtime behavior

```bash
kubectl get pods,svc,ingress
kubectl -n ingress-nginx port-forward service/ingress-nginx-controller 18080:80
```

In another terminal:

```bash
curl http://localhost:18080/api/data
curl http://localhost:18080/ | head -n 20
xdg-open http://localhost:18080/
```

## Troubleshooting (Production Flow)

### CI failed to push image

Check:
- DOCKER_HUB_TOKEN exists in repo secrets
- token has read/write scope
- Docker Hub namespace and repo are correct (mran66/k8s-test)

### Kubernetes ImagePullBackOff

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Common causes:
- CI did not publish expected tag
- wrong image name in deployment
- registry auth/network issue on cluster

### Ingress reachable issue

```bash
kubectl get ingress
kubectl describe ingress practice-app-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

## Rollback

```bash
kubectl rollout history deployment/backend-deployment
kubectl rollout history deployment/frontend-deployment

kubectl rollout undo deployment/backend-deployment
kubectl rollout undo deployment/frontend-deployment
```

## Cleanup

```bash
kubectl delete -f k8s-manifests/base/app-deployment-polyrepo.yaml \
  -f k8s-manifests/base/app-ingress.yaml
```
