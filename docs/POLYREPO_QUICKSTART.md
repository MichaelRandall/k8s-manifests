# Polyrepo Implementation Guide: Step-by-Step

Follow this guide to set up independent GitHub repositories for backend and frontend with separate CI/CD pipelines.

## Overview

```
Your Workflow:
┌─────────────────────────────────────────────────────────────┐
│ Development (Your Machine)                                  │
│ ├── ~/k8s-backend/ (git clone k8s-backend repo)             │
│ ├── ~/k8s-frontend/ (git clone k8s-frontend repo)           │
│ └── ~/k8s-manifests/ (git clone k8s-users repo)             │
└─────────────────────────────────────────────────────────────┘
                          ↓ git push
┌─────────────────────────────────────────────────────────────┐
│ GitHub CI/CD (Automated)                                    │
│ ├── k8s-backend repo → Actions → builds & pushes            │
│ ├── k8s-frontend repo → Actions → builds & pushes           │
│ └── k8s-users repo → (manifests only, no builds)            │
└─────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────┐
│ Docker Hub Registry                                         │
│ ├── m_ran66/k8s-backend:latest (updated independently)      │
│ └── m_ran66/k8s-frontend:latest (updated independently)     │
└─────────────────────────────────────────────────────────────┘
                          ↓ kubectl apply
┌─────────────────────────────────────────────────────────────┐
│ Kubernetes Cluster (Minikube)                               │
│ ├── Backend Pod (pulls from k8s-backend repo)               │
│ └── Frontend Pod (pulls from k8s-frontend repo)             │
└─────────────────────────────────────────────────────────────┘
```

## Phase 1: Create GitHub Repositories (5 minutes)

### 1.1 Create k8s-backend Repository

1. Go to https://github.com/new
2. Fill in:
   - Owner: MichaelRandall
   - Repository name: `k8s-backend`
   - Description: "Node.js backend service for Kubernetes"
   - Visibility: Public (or Private)
   - Initialize with README
3. Click **"Create repository"**
4. You'll see the repo at: `https://github.com/MichaelRandall/k8s-backend`

### 1.2 Create k8s-frontend Repository

1. Go to https://github.com/new
2. Fill in:
   - Owner: MichaelRandall
   - Repository name: `k8s-frontend`
   - Description: "Static HTML frontend for Kubernetes"
   - Visibility: Public (or Private)
   - Initialize with README
3. Click **"Create repository"**
4. You'll see the repo at: `https://github.com/MichaelRandall/k8s-frontend`

---

## Phase 2: Populate Repositories (10 minutes)

### 2.1 Set Up Backend Repository

```bash
# Clone the new backend repo
git clone https://github.com/MichaelRandall/k8s-backend.git
cd k8s-backend

# Copy backend files from your current monorepo
cp ~/Documents/self_directed/kubernetes_projects/my-kube/backend/Dockerfile .
cp ~/Documents/self_directed/kubernetes_projects/my-kube/backend/server.js .
cp ~/Documents/self_directed/kubernetes_projects/my-kube/backend/package.json .

# Create directory for workflow
mkdir -p .github/workflows

# Stage and commit
git add Dockerfile server.js package.json
git commit -m "Initial commit: Node.js backend service"
git push origin main
```

Your k8s-backend repo should now have:
```
k8s-backend/
├── Dockerfile
├── server.js
├── package.json
├── README.md
└── .github/
    └── workflows/  (created in Phase 3)
```

### 2.2 Set Up Frontend Repository

```bash
# Clone the new frontend repo
git clone https://github.com/MichaelRandall/k8s-frontend.git
cd k8s-frontend

# Copy frontend files from your current monorepo
cp ~/Documents/self_directed/kubernetes_projects/my-kube/frontend/Dockerfile .
cp ~/Documents/self_directed/kubernetes_projects/my-kube/frontend/index.html .

# Create directory for workflow
mkdir -p .github/workflows

# Stage and commit
git add Dockerfile index.html
git commit -m "Initial commit: Static HTML frontend"
git push origin main
```

Your k8s-frontend repo should now have:
```
k8s-frontend/
├── Dockerfile
├── index.html
├── README.md
└── .github/
    └── workflows/  (created in Phase 3)
```

---

## Phase 3: Add GitHub Actions Workflows (10 minutes)

### 3.1 Backend Repository Workflow

In your **k8s-backend** repo, create file `.github/workflows/build-and-push.yml`:

```bash
cd k8s-backend

# Create the workflow file
cat > .github/workflows/build-and-push.yml << 'EOF'
name: Build & Push Backend Image

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOCKER_HUB_USERNAME: m_ran66
  IMAGE_NAME: k8s-backend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Log in to Docker Hub
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          username: ${{ env.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_TOKEN }}
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
EOF

# Commit and push
git add .github/workflows/build-and-push.yml
git commit -m "Add GitHub Actions build workflow"
git push origin main
```

### 3.2 Frontend Repository Workflow

In your **k8s-frontend** repo, create file `.github/workflows/build-and-push.yml`:

```bash
cd k8s-frontend

# Create the workflow file (same structure, different IMAGE_NAME)
cat > .github/workflows/build-and-push.yml << 'EOF'
name: Build & Push Frontend Image

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  DOCKER_HUB_USERNAME: m_ran66
  IMAGE_NAME: k8s-frontend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Log in to Docker Hub
        if: github.event_name == 'push'
        uses: docker/login-action@v3
        with:
          username: ${{ env.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_TOKEN }}
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache,mode=max
EOF

# Commit and push
git add .github/workflows/build-and-push.yml
git commit -m "Add GitHub Actions build workflow"
git push origin main
```

---

## Phase 4: Add Docker Hub Secrets (5 minutes)

### 4.1 Add Secret to k8s-backend Repo

1. Go to: https://github.com/MichaelRandall/k8s-backend/settings/secrets/actions
2. Click **"New repository secret"**
3. Name: `DOCKER_HUB_TOKEN`
4. Value: (Paste your Docker Hub token from earlier)
5. Click **"Add secret"**

### 4.2 Add Secret to k8s-frontend Repo

1. Go to: https://github.com/MichaelRandall/k8s-frontend/settings/secrets/actions
2. Click **"New repository secret"**
3. Name: `DOCKER_HUB_TOKEN`
4. Value: (Same Docker Hub token)
5. Click **"Add secret"**

---

## Phase 5: Verify Builds (5-10 minutes)

### 5.1 Watch Backend Build

1. Go to: https://github.com/MichaelRandall/k8s-backend/actions
2. You should see "Build & Push Backend Image" workflow running
3. Wait for it to complete (2-5 minutes)
4. Check Docker Hub: https://hub.docker.com/r/m_ran66/k8s-backend
5. You should see `latest` tag and commit SHA tags

### 5.2 Watch Frontend Build

1. Go to: https://github.com/MichaelRandall/k8s-frontend/actions
2. You should see "Build & Push Frontend Image" workflow running
3. Wait for it to complete (2-5 minutes)
4. Check Docker Hub: https://hub.docker.com/r/m_ran66/k8s-frontend
5. You should see `latest` tag and commit SHA tags

---

## Phase 6: Update Kubernetes Deployment (5 minutes)

Now update your k8s-users repo (or create k8s-manifests folder) with the polyrepo manifest:

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube

# Add the polyrepo deployment file
# (already created as app-deployment-polyrepo.yaml)

# Or manually update your existing deployment:
# Change images from:
#   image: practice-backend:v5 (local)
#   imagePullPolicy: Never
# To:
#   image: m_ran66/k8s-backend:latest
#   imagePullPolicy: IfNotPresent

# Same for frontend:
#   image: m_ran66/k8s-frontend:latest
#   imagePullPolicy: IfNotPresent
```

---

## Phase 7: Deploy to Kubernetes (5 minutes)

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube

# Deploy using polyrepo manifest
kubectl apply -f app-deployment-polyrepo.yaml -f app-ingress.yaml

# Verify rollout
kubectl get pods
kubectl rollout status deployment/backend-deployment --timeout=180s
kubectl rollout status deployment/frontend-deployment --timeout=180s

# Access the app
kubectl -n ingress-nginx port-forward service/ingress-nginx-controller 18080:80

# In another terminal, open browser
open http://localhost:18080/
```

---

## Phase 8: Day-to-Day Development

### Backend Developer

```bash
cd ~/Documents/self_directed/kubernetes_projects/k8s-backend

# Make changes
nano server.js

# Test locally (optional)
npm install
npm start

# Commit and push
git add server.js
git commit -m "Add new API endpoint"
git push origin main

# GitHub Actions automatically:
# 1. Checks out code
# 2. Builds image: m_ran66/k8s-backend:latest
# 3. Pushes to Docker Hub
# (Watch in Actions tab)
```

### Frontend Developer (Simultaneously)

```bash
cd ~/Documents/self_directed/kubernetes_projects/k8s-frontend

# Make changes
nano index.html

# Commit and push
git add index.html
git commit -m "Update styling"
git push origin main

# GitHub Actions automatically:
# 1. Checks out code
# 2. Builds image: m_ran66/k8s-frontend:latest
# 3. Pushes to Docker Hub
# (Watch in Actions tab, completely independent from backend)
```

### Both Teams Update Kubernetes

When both images are ready:

```bash
# Redeploy (pulls latest images)
kubectl rollout restart deployment/backend-deployment
kubectl rollout restart deployment/frontend-deployment

# Or redeploy the manifest
kubectl apply -f app-deployment-polyrepo.yaml
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Workflow doesn't run | Make sure you pushed to `main` branch, not `develop` |
| Build fails | Check GitHub Actions logs: `Actions` tab → click failed workflow |
| Images don't appear on Docker Hub | Wait for workflow to finish, verify `DOCKER_HUB_TOKEN` secret exists |
| ImagePullBackOff in Kubernetes | Workflow may still running; wait 5 minutes, then redeploy |
| Can't access Docker Hub | Check token has Read & Write permissions |

---

## Summary: What You Now Have

✅ **Three independent GitHub repos:**
- k8s-backend (backend code + workflow)
- k8s-frontend (frontend code + workflow)
- k8s-users (Kubernetes manifests)

✅ **Two independent Docker Hub repositories:**
- m_ran66/k8s-backend (auto-built by backend repo)
- m_ran66/k8s-frontend (auto-built by frontend repo)

✅ **Independent development:**
- Backend Team can release v2.0 while Frontend is at v1.0
- Each team has its own workflow, release schedule
- Builds happen independently and in parallel

✅ **Kubernetes pulls both independently:**
- Can mix versions (backend v5, frontend v3)
- No synchronization needed

This is production-grade architecture used by Netflix, Uber, Stripe, etc.
