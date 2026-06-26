# GitHub Actions → Docker Hub → Kubernetes Workflow

This guide sets up a production-like CI/CD pipeline: code push → auto-build in GitHub Actions → push to Docker Hub → Kubernetes pulls from Docker Hub.

## Prerequisites

- GitHub repository: https://github.com/MichaelRandall/k8s-users
- Docker Hub account: https://hub.docker.com/ (free)
- Docker Hub Personal Access Token
- Kubernetes cluster (Minikube for testing)

---

## Step 1: Create Docker Hub Personal Access Token

1. Go to https://hub.docker.com/settings/security
2. Click **"New Access Token"**
3. Name it `GITHUB_ACTIONS` or similar
4. Permissions: Check **"Read & Write"**
5. Click **"Generate"**
6. **Copy the token immediately** (you won't see it again)

Example token looks like: `dckr_pat_abcd1234...`

---

## Step 2: Add Docker Hub Token to GitHub Secrets

1. Go to your GitHub repo: https://github.com/MichaelRandall/k8s-users
2. Settings → Secrets and variables → Actions
3. Click **"New repository secret"**
4. Name: `DOCKER_HUB_TOKEN`
5. Value: Paste your Docker Hub token
6. Click **"Add secret"**

Now GitHub Actions can authenticate to Docker Hub without exposing your password.

---

## Step 3: Push Workflow to GitHub

The workflow file is already created at `.github/workflows/build-and-push.yml`

Push to GitHub:

```bash
cd /path/to/my-kube

git add .github/workflows/build-and-push.yml
git commit -m "Add GitHub Actions workflow for Docker Hub builds"
git push origin main
```

---

## Step 4: Watch the Build

1. Go to your GitHub repo
2. Click **"Actions"** tab
3. You'll see the workflow running

The workflow will:
- Trigger on push to `main` branch
- Build backend image: `mran66/k8s-test:backend-latest`
- Build frontend image: `mran66/k8s-test:frontend-latest`
- Push both to Docker Hub
- Add git commit SHA as a tag (e.g., `mran66/k8s-back:main-abc123def`)

Check Docker Hub after the workflow completes:
https://hub.docker.com/r/mran66/k8s-back

You should see your images listed there.

---

## Step 5: Deploy to Kubernetes from Docker Hub

Now your Kubernetes cluster can pull images directly from Docker Hub instead of building locally.

### Option A: Use the Docker Hub Deployment Manifest

```bash
cd /path/to/my-kube

# Use the Docker Hub manifest instead of the local build one
kubectl apply -f k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml -f k8s-manifests/base/app-ingress.yaml

# Verify rollout
kubectl rollout status deployment/backend-deployment
kubectl rollout status deployment/frontend-deployment

# Access the app
kubectl -n ingress-nginx port-forward service/ingress-nginx-controller 18080:80
xdg-open http://localhost:18080/
```

### Option B: Update Your Current Manifest

If you prefer to keep using `app-deployment.yaml`, just change the image references:

```yaml
# In app-deployment.yaml, replace:
# FROM: image: practice-backend:v5, imagePullPolicy: Never
# TO:   image: mran66/k8s-test:backend-latest, imagePullPolicy: IfNotPresent
```

---

## How It Works: The Full Flow

```
1. Developer pushes code to GitHub (main branch)
   └─ git push origin main

2. GitHub Actions workflow triggered automatically
   ├─ Checks out code
   ├─ Logs in to Docker Hub with DOCKER_HUB_TOKEN secret
   ├─ Builds backend image: mran66/k8s-test:backend-latest (and :main-abc123)
   ├─ Pushes to Docker Hub
   ├─ Builds frontend image: mran66/k8s-test:frontend-latest (and :main-abc123)
   ├─ Pushes to Docker Hub
   └─ Workflow completes (visible in GitHub Actions tab)

3. Operator deploys to Kubernetes
   └─ kubectl apply -f k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml

4. Kubernetes pulls images from Docker Hub
   ├─ kubelet sees: image: mran66/k8s-test:backend-latest
   ├─ Checks imagePullPolicy: IfNotPresent
   ├─ Pulls from docker.io/mran66/k8s-test:backend-latest
   ├─ Starts container
   └─ Pod becomes Ready

5. App is live, no local build needed!
```

---

## Key Differences from Local Minikube Build

| Aspect | Local (minikube image build) | Docker Hub (this workflow) |
|--------|------------------------------|---------------------------|
| **Build location** | Your machine | GitHub Actions cloud |
| **Image storage** | Local Minikube Docker | Docker Hub registry |
| **Trigger** | Manual (`minikube image build`) | Automatic on git push |
| **Access** | Minikube only | Any K8s cluster |
| **Reproducibility** | Dev-specific | Git commit SHA |
| **Speed** | Fast (local) | ~2-5 minutes (network) |
| **Team collaboration** | Must manually share images | Everyone pulls from same registry |

---

## Workflow Tags Explained

The GitHub Actions workflow generates multiple tags for each push:

```bash
# For a push to main branch with git commit abc123:
mran66/k8s-back:main              # Branch tag
mran66/k8s-back:main-abc123       # Commit SHA tag (unique)
mran66/k8s-test:backend-latest            # Latest on main
```

**Use in production:**

```yaml
# Development: Always pull latest
image: mran66/k8s-test:backend-latest

# Production: Pin to specific commit for reproducibility
image: mran66/k8s-back:main-abc123def456
```

---

## Troubleshooting

### Workflow failed: "Error logging in to Docker Hub"

**Cause**: `DOCKER_HUB_TOKEN` secret not set or incorrect.

**Fix**:
1. Go to GitHub Settings → Secrets and variables → Actions
2. Verify `DOCKER_HUB_TOKEN` exists and is correct
3. Re-run the workflow

### Images don't appear on Docker Hub

**Cause**: Workflow only pushes on actual `push` events, not on pull requests.

**Verify**:
- You pushed to `main` branch (not a PR)
- Workflow ran successfully (check Actions tab)
- Check Docker Hub: https://hub.docker.com/r/mran66

### Kubernetes can't pull image

**Cause**: Image doesn't exist, or Kubernetes can't reach Docker Hub.

**Debug**:

```bash
# Check if image exists on Docker Hub
curl -s https://registry.hub.docker.com/v2/mran66/k8s-back/tags/list

# Check pod events
kubectl describe pod <pod-name>

# Check pod logs
kubectl logs <pod-name>

# If stuck in ImagePullBackOff, wait for workflow to complete and push
```

### Want to make images private?

1. On Docker Hub, go to repository settings
2. Set to **Private**
3. Create a Docker Hub credential in Kubernetes:

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=docker.io \
  --docker-username=mran66 \
  --docker-password=<your-password-or-token> \
  --docker-email=your@email.com
```

Then in deployment YAML:

```yaml
spec:
  imagePullSecrets:
  - name: dockerhub-secret
  containers:
  - name: backend
    image: mran66/k8s-test:backend-latest
```

---

## Next Steps

1. ✅ Create Docker Hub token
2. ✅ Add `DOCKER_HUB_TOKEN` to GitHub secrets
3. ✅ Push `.github/workflows/build-and-push.yml` to GitHub
4. ✅ Watch workflow build your images
5. ✅ Deploy using `k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml`
6. Test: Update code, push, watch GitHub Actions build automatically
7. Optional: Add image scanning (Snyk), tests, security checks to workflow

---

## Real-World Extensions

Once this works, you can add:

- **Tests**: Run unit tests, linting before building images
- **Image scanning**: Check for vulnerabilities (Snyk, Trivy)
- **Deployment automation**: Auto-deploy to staging on push, manually approve for prod
- **GitOps**: Use ArgoCD to watch your git repo and auto-sync to K8s
- **Release tracking**: Tag releases separately, deploy specific versions to prod
- **Multi-environment**: Different workflows for dev, staging, production clusters

Example enhanced workflow structure:

```
.github/workflows/
├── build-and-push.yml      # (current) Build on every push
├── deploy-staging.yml      # Auto-deploy to staging
├── deploy-prod.yml         # Manual approval for production
└── security-scan.yml       # Scan images for vulnerabilities
```

This is how companies like Shopify, Stripe, Netflix manage thousands of services.
