# Polyrepo Setup: Frontend & Backend as Separate GitHub Repositories

This guide converts your monorepo into a polyrepo structure where frontend and backend are independently developed, built, and deployed.

## Architecture Overview

```
Development Phase:
├── Developer 1: Works on backend/ → Pushes to k8s-backend repo
└── Developer 2: Works on frontend/ → Pushes to k8s-frontend repo

CI/CD Phase:
├── k8s-backend repo → GitHub Actions → Docker Hub (m_ran66/k8s-backend:latest)
└── k8s-frontend repo → GitHub Actions → Docker Hub (m_ran66/k8s-frontend:latest)

Deployment Phase:
Kubernetes pulls both images independently:
├── k8s-backend:v5 (latest from backend repo)
└── k8s-frontend:v3 (latest from frontend repo)
```

## Step 1: Create GitHub Repositories

### Create Backend Repository

1. Go to https://github.com/new
2. Repository name: `k8s-backend`
3. Description: "Kubernetes backend service (Node.js + Express)"
4. Public or Private (your choice)
5. Initialize with: README
6. Click **"Create repository"**
7. Copy the HTTPS URL: `https://github.com/MichaelRandall/k8s-backend`

### Create Frontend Repository

1. Go to https://github.com/new
2. Repository name: `k8s-frontend`
3. Description: "Kubernetes frontend service (Static HTML)"
4. Public or Private (your choice)
5. Initialize with: README
6. Click **"Create repository"**
7. Copy the HTTPS URL: `https://github.com/MichaelRandall/k8s-frontend`

## Step 2: Populate Repositories (From Your Local Monorepo)

### Backend Repository

```bash
# Clone the new backend repo
git clone https://github.com/MichaelRandall/k8s-backend.git
cd k8s-backend

# Copy backend files from your monorepo
cp -r ~/Documents/self_directed/kubernetes_projects/my-kube/backend/* .

# Should now have:
# ├── Dockerfile
# ├── server.js
# ├── package.json
# └── README.md (from GitHub)

# Create GitHub Actions workflow directory
mkdir -p .github/workflows

# (We'll add the workflow file in Step 3)

git add .
git commit -m "Initial commit: Node.js backend service"
git push origin main
```

### Frontend Repository

```bash
# Clone the new frontend repo
git clone https://github.com/MichaelRandall/k8s-frontend.git
cd k8s-frontend

# Copy frontend files from your monorepo
cp -r ~/Documents/self_directed/kubernetes_projects/my-kube/frontend/* .

# Should now have:
# ├── Dockerfile
# ├── index.html
# └── README.md (from GitHub)

# Create GitHub Actions workflow directory
mkdir -p .github/workflows

# (We'll add the workflow file in Step 3)

git add .
git commit -m "Initial commit: Static HTML frontend"
git push origin main
```

## Step 3: Add GitHub Actions Workflows to Each Repo

### Backend Repository Workflow

Create `.github/workflows/build-and-push.yml` in k8s-backend repo:

```yaml
name: Build & Push Backend Image

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  DOCKER_HUB_USERNAME: m_ran66
  IMAGE_NAME: k8s-backend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

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

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

      - name: Image digest
        run: |
          echo "Image: docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}"
          echo "Tags: ${{ steps.meta.outputs.tags }}"
```

### Frontend Repository Workflow

Create `.github/workflows/build-and-push.yml` in k8s-frontend repo:

```yaml
name: Build & Push Frontend Image

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

env:
  DOCKER_HUB_USERNAME: m_ran66
  IMAGE_NAME: k8s-frontend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

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

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name == 'push' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

      - name: Image digest
        run: |
          echo "Image: docker.io/${{ env.DOCKER_HUB_USERNAME }}/${{ env.IMAGE_NAME }}"
          echo "Tags: ${{ steps.meta.outputs.tags }}"
```

## Step 4: Add Secrets to Both Repos

**For k8s-backend repo:**
1. Go to https://github.com/MichaelRandall/k8s-backend/settings/secrets/actions
2. Click **"New repository secret"**
3. Name: `DOCKER_HUB_TOKEN`
4. Value: (same token as before)
5. Click **"Add secret"**

**For k8s-frontend repo:**
1. Go to https://github.com/MichaelRandall/k8s-frontend/settings/secrets/actions
2. Click **"New repository secret"**
3. Name: `DOCKER_HUB_TOKEN`
4. Value: (same token as before)
5. Click **"Add secret"**

## Step 5: Update Local Development Structure

Keep your local folder the same, but now it acts as a **workspace** for both repos:

```bash
~/Documents/self_directed/kubernetes_projects/my-kube/
├── backend/              # Linked to k8s-backend repo
│   ├── .git
│   ├── Dockerfile
│   ├── server.js
│   └── package.json
├── frontend/             # Linked to k8s-frontend repo
│   ├── .git
│   ├── Dockerfile
│   └── index.html
└── k8s-manifests/        # (NEW) Kubernetes deployments
    ├── app-deployment-polyrepo.yaml
    └── app-ingress.yaml
```

### Set Up Git Submodules (Optional, Advanced)

If you want to manage both repos as submodules in your local workspace:

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube

# Remove the old backend and frontend directories
rm -rf backend frontend

# Add them as git submodules
git submodule add https://github.com/MichaelRandall/k8s-backend.git backend
git submodule add https://github.com/MichaelRandall/k8s-frontend.git frontend

# Commit
git add .gitmodules backend frontend
git commit -m "Add backend and frontend as git submodules"
git push origin main
```

Now when you do:
```bash
git clone https://github.com/MichaelRandall/k8s-users.git
cd k8s-users
git submodule update --init --recursive
```

Both repos are cloned automatically.

### (Or) Keep Separate Clones

Simpler approach: just clone each repo separately in your workspace:

```bash
cd ~/Documents/self_directed/kubernetes_projects/

# Clone both repos
git clone https://github.com/MichaelRandall/k8s-backend.git
git clone https://github.com/MichaelRandall/k8s-frontend.git
git clone https://github.com/MichaelRandall/k8s-users.git k8s-manifests  # For Kubernetes files
```

## Step 6: Create Polyrepo Kubernetes Manifest

In your k8s-users repo (or k8s-manifests folder), create `app-deployment-polyrepo.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: m_ran66/k8s-backend:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
        readinessProbe:
          httpGet:
            path: /api/data
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/data
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: m_ran66/k8s-frontend:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 150m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

**Key differences from monorepo:**
- Images pull independently from Docker Hub
- Backend can be version v5, Frontend can be version v3
- Versions are not synchronized
- Workflows trigger independently

## Step 7: Deploy to Kubernetes

```bash
# Deploy from polyrepo manifest
kubectl apply -f k8s-manifests/app-deployment-polyrepo.yaml \
              -f k8s-manifests/app-ingress.yaml

# Verify
kubectl get pods
kubectl rollout status deployment/backend-deployment
kubectl rollout status deployment/frontend-deployment
```

## Development Workflow: Day-to-Day

### Backend Developer

```bash
cd ~/Documents/self_directed/kubernetes_projects/k8s-backend

# Make changes
nano server.js

# Commit and push
git add server.js
git commit -m "Add new API endpoint"
git push origin main

# GitHub Actions automatically:
# - Builds m_ran66/k8s-backend:latest
# - Builds m_ran66/k8s-backend:main-abc123
```

### Frontend Developer (Simultaneously)

```bash
cd ~/Documents/self_directed/kubernetes_projects/k8s-frontend

# Make changes
nano index.html

# Commit and push
git add index.html
git commit -m "Update UI styling"
git push origin main

# GitHub Actions automatically:
# - Builds m_ran66/k8s-frontend:latest
# - Builds m_ran66/k8s-frontend:main-def456
```

### Both Can Release Independently

```bash
# Backend releases v2.0.0
cd k8s-backend
git tag v2.0.0
git push origin v2.0.0
# Workflow creates: m_ran66/k8s-backend:2.0.0

# Frontend still on v1.0.0 (no changes yet)
# Production continues running:
# - Backend v2.0.0
# - Frontend v1.0.0
```

## Advantages of Polyrepo

✅ **Independent development**: No waiting for other team  
✅ **Independent releases**: Backend v5 while Frontend v2  
✅ **Clear ownership**: Frontend team owns frontend repo  
✅ **Smaller repos**: Faster clones, easier to understand  
✅ **Parallel CI/CD**: Both build simultaneously  
✅ **Separate workflows**: Each service can have custom tests, rules  

## Disadvantages of Polyrepo

❌ **Coordination complexity**: More moving parts  
❌ **Harder to keep in sync**: Can diverge more easily  
❌ **More repositories to manage**: 2 instead of 1  
❌ **Breaking changes harder**: API changes need compatibility  

## When to Switch Back to Monorepo

If you find polyrepo getting complex:
- Merge repos back into k8s-users
- Single workflow, single release
- Easier for small teams

## Validation Checklist

- [ ] k8s-backend repo created with Dockerfile, server.js, package.json
- [ ] k8s-frontend repo created with Dockerfile, index.html
- [ ] GitHub Actions workflows added to both repos
- [ ] `DOCKER_HUB_TOKEN` secret added to both repos
- [ ] Repos pushed to main branch
- [ ] Workflows completed on GitHub Actions
- [ ] Images appear on Docker Hub
- [ ] Kubernetes deployment pulls both images successfully
- [ ] App is live at http://localhost:18080/

You now have a production-grade polyrepo setup!
