# Quick Start & Operations Runbook

Practical commands for your Minikube app workflow.

## Prerequisites

- Minikube installed and running: `minikube status`
- kubectl installed: `kubectl version --client`
- Docker CLI available (for local builds): `docker --version`

---

## Initial Setup (One Time)

```bash
# Start Minikube cluster
minikube start

# Enable required addons
minikube addons enable ingress
minikube addons enable ingress-dns

# Verify cluster is ready
kubectl cluster-info
kubectl get nodes
```

---

## Build & Deploy Workflow

### Option A: Local Build (Fast, Dev Only)

Build directly into Minikube for quick iteration:

```bash
cd /path/to/my-kube

# Build backend image
minikube image build -t practice-backend:v5 ./backend

# Build frontend image
minikube image build -t practice-frontend:v2 ./frontend

# Deploy using local images
kubectl apply -f k8s-manifests/overlays/local/app-deployment.yaml -f k8s-manifests/base/app-ingress.yaml
```

**When to use:**
- Local development
- Quick testing
- Learning/experimentation

**Limitations:**
- Only works on your machine
- Not reproducible across team
- Images not shared

### Option B: Docker Hub (Production-like, Team-friendly)

Automatically build images in GitHub Actions and push to Docker Hub:

```bash
# 1. Make code changes
nano backend/server.js

# 2. Commit and push to GitHub
git add .
git commit -m "Update API endpoint"
git push origin main

# 3. GitHub Actions automatically:
#    - Builds backend image: mran66/k8s-test:backend-latest
#    - Builds frontend image: mran66/k8s-test:frontend-latest
#    - Pushes to Docker Hub
#    (Check: GitHub repo → Actions tab)

# 4. Deploy from Docker Hub
kubectl apply -f k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml -f k8s-manifests/base/app-ingress.yaml

# 5. Kubernetes automatically pulls from Docker Hub when rolling out
```

### Option B Rollout (Current Working Commands)

Use this exact sequence after both GitHub Actions builds are green and tags exist in Docker Hub:

```bash
cd ~/Documents/self_directed/kubernetes_projects/my-kube

# Apply manifests
kubectl apply -f k8s-manifests/base/app-deployment-polyrepo.yaml -f k8s-manifests/base/app-ingress.yaml

# Force pods to pull latest backend/frontend tags
kubectl rollout restart deployment/backend-deployment
kubectl rollout restart deployment/frontend-deployment

# Wait until both are ready
kubectl rollout status deployment/backend-deployment --timeout=180s
kubectl rollout status deployment/frontend-deployment --timeout=180s

# Verify running images
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
# Expected: mran66/k8s-test:backend-latest mran66/k8s-test:frontend-latest

# Expose ingress locally
kubectl -n ingress-nginx port-forward service/ingress-nginx-controller 18080:80
```

**When to use:**
- Team development (share images)
- Production deployments
- CI/CD pipeline learning
- Any multi-machine setup

**First-time setup for Docker Hub workflow:**
1. See `GITHUB_ACTIONS_SETUP.md` for detailed instructions
2. Create Docker Hub Personal Access Token
3. Add `DOCKER_HUB_TOKEN` secret to GitHub
4. Push `.github/workflows/build-and-push.yml` to GitHub
5. Future pushes to `main` automatically build and push images

### 3. Verify Rollout

```bash
# Wait for backend deployment to be ready (blocks until complete)
kubectl rollout status deployment/backend-deployment --timeout=180s

# Wait for frontend deployment
kubectl rollout status deployment/frontend-deployment --timeout=180s

# Quick status check
kubectl get pods,svc,ingress
```

**For Docker Hub deployments:**

If using `k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml`, Kubernetes will pull images from Docker Hub. Watch the pod status:

```bash
# Pod will transition: Pending → ContainerCreating → Running
kubectl get pods -w

# If stuck in ImagePullBackOff, check events
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Verify image was pulled
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'
# Output: mran66/k8s-test:backend-latest mran66/k8s-test:frontend-latest
```

### 4. Access Application

#### Option A: Port Forward (Recommended for Local Dev)

```bash
# In one terminal, keep this running
kubectl -n ingress-nginx port-forward service/ingress-nginx-controller 18080:80

# In another terminal, open browser
xdg-xdg-open http://localhost:18080/
```

**Why port-forward?**
- Works reliably across different Minikube driver configs
- No host networking issues
- Easy debugging (request goes: Browser → port-forward → Ingress Controller → Service → Pods)

#### Option B: Minikube Tunnel (Persistent)

```bash
# In one terminal, keep this running
minikube tunnel

# In another terminal
xdg-xdg-open http://$(minikube ip)/
```

**Note**: Requires sudo, may ask for your password periodically.

#### Option C: NodePort (Direct, Less Recommended)

```bash
# Get Minikube IP
minikube ip  # E.g., 192.168.49.2

# Access via explicit node port
xdg-open http://192.168.49.2:30002/  # Frontend (if using NodePort service)
```

---

## Verify Everything Works

### Test Backend API Directly

```bash
# Via port-forward
curl http://localhost:18080/api/data

# Expected output:
# [{"id":1,"name":"Alice",...}, ...]
```

### Test Frontend

```bash
curl http://localhost:18080/ | head -n 20

# Expected: HTML with user list in script
```

### Test from Within Cluster

```bash
# Launch temporary pod with curl
kubectl run curl-test --rm -i --restart=Never \
  --image=curlimages/curl:8.8.0 -- \
  curl http://backend-service:3000/api/data

# This tests internal DNS and service routing
```

---

## Common Operations

### View Pod Logs

```bash
# Get pod name
kubectl get pods

# View logs (last 100 lines)
kubectl logs backend-deployment-c64f846f5-g6m4l

# Stream logs in real-time
kubectl logs -f backend-deployment-c64f846f5-g6m4l

# View logs from all pods in deployment
kubectl logs -f deployment/backend-deployment
```

### Shell Into Pod

```bash
# Get shell access to pod for debugging
kubectl exec -it backend-deployment-c64f846f5-g6m4l -- sh

# Inside the pod, you can run commands
curl http://localhost:3000/api/data
curl http://frontend-service:80/
ps aux
env
```

### View Resource Usage

```bash
# CPU and memory per pod
kubectl top pods

# CPU and memory per node
kubectl top nodes

# (Requires metrics-server, usually installed on production clusters)
```

### Describe Pod (Detailed Info)

```bash
kubectl describe pod backend-deployment-c64f846f5-g6m4l

# Shows: image, port, probes, events, status
# Useful for debugging why pod won't start
```

### View Service Details

```bash
kubectl describe svc backend-service

# Shows: endpoints (which pods backing this service), port mappings, etc.
```

### View Ingress Details

```bash
kubectl describe ingress practice-app-ingress

# Shows: routing rules, backend services, status
```

---

## Update Workflow

### Update Backend Code

```bash
# Edit backend/server.js
nano backend/server.js

# Rebuild image (bump version)
minikube image build -t practice-backend:v6 ./backend

# Update deployment to use new image
# Edit app-deployment.yaml, change image: practice-backend:v6
nano app-deployment.yaml

# Apply changes
kubectl apply -f app-deployment.yaml

# Watch rollout
kubectl rollout status deployment/backend-deployment

# Verify new pod is running
kubectl get pods
kubectl logs -f deployment/backend-deployment
```

### Rollback to Previous Version

```bash
# View rollout history
kubectl rollout history deployment/backend-deployment

# Revision 1: image v4
# Revision 2: image v5
# Revision 3: image v6 (current)

# Revert to previous revision
kubectl rollout undo deployment/backend-deployment

# Revert to specific revision
kubectl rollout undo deployment/backend-deployment --to-revision=1
```

---

## Scaling

### Increase Replicas

```bash
# Update deployment to run 3 copies
kubectl scale deployment/backend-deployment --replicas=3

# Verify
kubectl get pods

# Now 3 backend pods running, service load-balances between them
```

### Permanently Scale

```bash
# Edit deployment YAML
nano k8s-manifests/overlays/local/app-deployment.yaml
# Change: spec.replicas: 3

# Apply
kubectl apply -f k8s-manifests/overlays/local/app-deployment.yaml
```

---

## Clean Up

### Delete Everything

```bash
# Delete all resources defined in manifests
kubectl delete -f k8s-manifests/base/app-deployment-polyrepo.yaml -f k8s-manifests/base/app-ingress.yaml

# Or specific resource
kubectl delete deployment backend-deployment
kubectl delete svc backend-service
```

### Stop Minikube

```bash
# Gracefully stop cluster (preserves data)
minikube stop

# Delete cluster (removes everything, free up disk)
minikube delete
```

---

## Troubleshooting

### GitHub Actions Workflow Issues

| Problem | Check |
|---------|-------|
| Workflow not triggered | Verify push to `main` branch (other branches don't trigger) |
| Build failed | GitHub repo → Actions tab, click workflow → view logs |
| Images not on Docker Hub | Visit https://hub.docker.com/r/mran66 and refresh |
| DOCKER_HUB_TOKEN error | GitHub Settings → Secrets and variables → verify `DOCKER_HUB_TOKEN` exists |
| ImagePullBackOff in K8s | Workflow may still building; wait 2-5 minutes, then pull latest |

### Pod Stuck in Pending

```bash
kubectl describe pod <pod-name>

# Check: Image pull errors, resource requests too high, no available nodes
```

### Pod Crashes Immediately

```bash
kubectl logs <pod-name>           # View startup logs
kubectl describe pod <pod-name>   # Check events

# Common: wrong image tag, missing dependencies, port already in use
```

### Service Has No Endpoints

```bash
kubectl get endpoints backend-service

# If empty, service selector doesn't match any pods
# Verify pod labels match service selector in YAML
```

### Ingress Not Routing

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Check: ingress rules are correct, services/ports exist
```

### Port-Forward Hangs

```bash
# Kill the process
Ctrl+C

# Try again, or use minikube tunnel instead
minikube tunnel
```

---

## Debugging Checklist

| Problem | Check |
|---------|-------|
| Pod won't start | `kubectl describe pod`, `kubectl logs` |
| Can't reach app | `kubectl get svc`, `kubectl port-forward` working? |
| Data not loading | `curl localhost:18080/api/data`, check backend logs |
| Service discovery fails | `kubectl exec -it <pod> -- curl backend-service:3000` |
| Image not found | `minikube image ls`, rebuild image |
| Port already in use | `lsof -i :18080`, kill process, try different port |

---

## Environment Variables (Reference)

To pass config without rebuilding images:

```yaml
# In deployment YAML
containers:
- name: app
  env:
  - name: LOG_LEVEL
    value: "debug"
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: api-key
```

---

## File Structure

```
my-kube/
├── k8s-manifests/
│   ├── base/
│   │   ├── app-deployment-polyrepo.yaml
│   │   └── app-ingress.yaml
│   └── overlays/
│       ├── local/app-deployment.yaml
│       └── dockerhub/app-deployment-dockerhub.yaml
├── docs/
│   ├── RUNBOOK.md
│   └── TRAINING_GUIDE.md
├── templates/
│   └── github-actions/
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   └── server.js
├── frontend/
│   ├── Dockerfile
│   └── index.html
└── README.md
```

---

## Next Steps

1. **Learn**: Read TRAINING_GUIDE.md for concepts
2. **Experiment**: Try `kubectl scale`, `kubectl exec`, `kubectl logs`
3. **Modify**: Change `server.js` endpoint, rebuild, redeploy
4. **Monitor**: Add CPU/memory limits, watch resource usage
5. **Advance**: Add ConfigMaps, Secrets, NetworkPolicies, RBAC

---

## Useful Links

- Kubernetes Docs: https://kubernetes.io/docs/
- kubectl Cheatsheet: https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- Minikube Docs: https://minikube.sigs.k8s.io/
- YAML Validator: https://www.yamllint.com/
