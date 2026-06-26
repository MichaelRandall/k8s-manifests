# Get Started with GitHub Actions → Docker Hub → Kubernetes

This is your step-by-step checklist to switch from local Minikube builds to a production-like CI/CD workflow.

## 5-Minute Setup

### Step 1: Create Docker Hub Token (1 min)

1. Go to https://hub.docker.com/settings/security
2. Click **"New Access Token"**
3. Name it `GITHUB_ACTIONS`
4. Check **"Read & Write"**
5. Click **"Generate"**
6. **Copy the token** (you won't see it again)

### Step 2: Add Token to GitHub (1 min)

1. Go to https://github.com/MichaelRandall/k8s-users
2. Settings → Secrets and variables → Actions
3. Click **"New repository secret"**
4. Name: `DOCKER_HUB_TOKEN`
5. Value: Paste the token from Step 1
6. Click **"Add secret"**

### Step 3: Push Workflow to GitHub (2 min)

The workflow file already exists at `.github/workflows/build-and-push.yml`. Push it:

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube

git add .github/workflows/build-and-push.yml .github/workflows/.gitkeep 2>/dev/null || true
git add GITHUB_ACTIONS_SETUP.md k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml
git commit -m "Add GitHub Actions CI/CD workflow for Docker Hub"
git push origin main
```

### Step 4: Watch the Build (2 min)

1. Go to https://github.com/MichaelRandall/k8s-users/actions
2. You'll see the workflow running
3. Wait for it to complete (usually 2-5 minutes)
4. Check Docker Hub: https://hub.docker.com/r/mran66
5. You should see `k8s-back` and `k8s-front` repositories with images

### Step 5: Deploy to Kubernetes (1 min)

```bash
# Use Docker Hub images instead of local builds
kubectl apply -f k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml -f k8s-manifests/base/app-ingress.yaml

# Watch it pull and start
kubectl get pods -w

# Verify
kubectl rollout status deployment/backend-deployment
kubectl rollout status deployment/frontend-deployment

# Access
kubectl -n ingress-nginx port-forward service/ingress-nginx-controller 18080:80
xdg-open http://localhost:18080/
```

**Done!** You now have a production-like CI/CD pipeline.

---

## What Just Happened?

```
Your Local Machine                   GitHub Cloud                    Docker Hub           Kubernetes Cluster
─────────────────────────────────────────────────────                ──────────          ──────────────────

git push origin main    ──────────→  GitHub Actions workflow
                                      ├─ Checks out code
                                      ├─ Builds images
                                      ├─ Logs in with DOCKER_HUB_TOKEN
                                      ├─ Pushes to Docker Hub  ─────→  Stores images
                                      └─ Workflow complete              │
                                                                        │
kubectl apply -f      ─────────────────────────────────────────────────→ Pulls images
k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml                                            from Docker Hub
                                                                         │
                                                         Kubernetes runs ──┘
                                                         your app from
                                                         Docker Hub images
```

No local Docker build needed. No manual image management. Just git push and Kubernetes deploys.

---

## Common Next Steps

### Update Code & Auto-Deploy

```bash
# 1. Make a change
nano backend/server.js
# Change a console.log or API response

# 2. Commit and push
git add backend/server.js
git commit -m "Update API response"
git push origin main

# 3. GitHub Actions automatically:
#    - Builds new image
#    - Pushes to Docker Hub
#    (Check GitHub Actions tab)

# 4. When ready, redeploy Kubernetes (picks up new image)
kubectl rollout restart deployment/backend-deployment
# (or wait for imagePullPolicy to pull new image on next pod restart)
```

### Pin to Specific Versions

Instead of `latest`, use git commit SHAs for production:

```bash
# The workflow also creates tags with commit SHA
# Example: k8s-back:main-abc123def456

# In production deployment YAML:
image: mran66/k8s-back:main-abc123def456
imagePullPolicy: IfNotPresent
```

This ensures you deploy exactly the code you tested.

### Add More Workflows

`.github/workflows/` can have multiple workflows:

```
.github/workflows/
├── build-and-push.yml         # Current: builds on every push
├── deploy-staging.yml         # Deploy to staging after build
├── security-scan.yml          # Scan images for vulnerabilities
└── deploy-prod-manual.yml     # Manual approval for production
```

---

## Troubleshooting Quick Fixes

| Issue | Fix |
|-------|-----|
| "DOCKER_HUB_TOKEN not found" | Go to GitHub Settings → Secrets and add it |
| Images not on Docker Hub | Check GitHub Actions tab for workflow errors |
| Pod stuck in ImagePullBackOff | Wait for workflow to finish (2-5 min), then redeploy |
| Workflow won't trigger | Make sure you pushed to `main` branch, not just `main-local` |
| Can't access Docker Hub | Verify token has Read & Write permissions |

---

## Reference Docs

- **Full Setup Guide**: See `GITHUB_ACTIONS_SETUP.md`
- **Docker/Kubernetes Concepts**: See `TRAINING_GUIDE.md`
- **All Commands**: See `RUNBOOK.md`

---

## You Now Have

✅ Automated image builds via GitHub Actions  
✅ Images stored on Docker Hub (accessible anywhere)  
✅ Kubernetes pulling from Docker Hub (production-style)  
✅ Zero manual Docker commands needed  
✅ Reproducible builds (tied to git commits)  
✅ Team-friendly (images shared, not machine-specific)  

This is how real teams do it. Congratulations!
