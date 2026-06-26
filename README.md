# Local Polyrepo Workspace

This workspace is organized for polyrepo-style development.

## Repositories

- `backend/` -> backend service repository (`k8s-back`)
- `frontend/` -> frontend service repository (`k8s-front`)
- `k8s-manifests/` -> Kubernetes manifests repository content

## Kubernetes Layout

- `k8s-manifests/base/`
  - `app-deployment-polyrepo.yaml`
  - `app-ingress.yaml`
- `k8s-manifests/overlays/local/`
  - `app-deployment.yaml` (local/minikube image workflow)
- `k8s-manifests/overlays/dockerhub/`
  - `app-deployment-dockerhub.yaml` (registry pull workflow)

## Supporting Material

- `docs/` -> training, setup, and runbook guides
- `templates/github-actions/` -> reusable workflow templates for service repos

## Suggested Next Step

Create a dedicated GitHub repository for the manifests and push only `k8s-manifests/` (plus optional docs relevant to deployment).
