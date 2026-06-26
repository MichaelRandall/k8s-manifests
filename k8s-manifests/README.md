# Kubernetes Manifests

This folder contains deployment manifests separated by use case.

## Base

- `base/app-deployment-polyrepo.yaml` -> canonical deployment pulling service images from Docker Hub
- `base/app-ingress.yaml` -> ingress routing for frontend and backend services

## Overlays

- `overlays/local/app-deployment.yaml` -> local/minikube workflow using local image tags
- `overlays/dockerhub/app-deployment-dockerhub.yaml` -> dockerhub-focused deployment variant

## Apply Commands

Polyrepo base deployment:

```bash
kubectl apply -f k8s-manifests/base/app-deployment-polyrepo.yaml \
              -f k8s-manifests/base/app-ingress.yaml
```

Local workflow deployment:

```bash
kubectl apply -f k8s-manifests/overlays/local/app-deployment.yaml \
              -f k8s-manifests/base/app-ingress.yaml
```

Docker Hub overlay deployment:

```bash
kubectl apply -f k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml \
              -f k8s-manifests/base/app-ingress.yaml
```
