# Docker & Kubernetes Training: Your Application

This guide teaches Docker and Kubernetes concepts using your frontend/backend Minikube application as the teaching vehicle.

---

## Part 1: Docker Fundamentals

### 1.1 What is Docker?

Docker is a containerization platform that packages your application code, dependencies, runtime, and configuration into a lightweight, portable, isolated unit called a **container**. Think of it like shipping containers for software: standardized, portable across environments, and reproducible.

**Why Docker?**
- **Consistency**: "It works on my machine" → "It works everywhere" 
- **Isolation**: Containers don't interfere with each other or the host
- **Efficiency**: Lightweight (MB-scale) vs full VMs (GB-scale)
- **Speed**: Containers start in seconds, not minutes

### 1.2 Docker Images vs Containers

**Image**: A read-only blueprint for a container. Think of it as a class definition.
- Contains: OS base layer, app code, dependencies, environment config
- Static, versioned (tagged with version numbers like `v1`, `v5`)
- Built once, reused many times

**Container**: A running instance of an image. Think of it as an object instantiated from a class.
- Isolated runtime environment
- Can be started, stopped, deleted
- Changes to a container are ephemeral (lost when container stops)

### 1.3 The Dockerfile

A Dockerfile is a recipe for building an image. Let's examine your backend Dockerfile:

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev
COPY server.js ./
EXPOSE 3000
CMD ["npm", "start"]
```

**Line-by-line breakdown:**

| Line | Instruction | Purpose |
|------|-----------|---------|
| `FROM node:18-alpine` | Base image | Start with Node.js 18 on Alpine Linux (minimal, ~160MB) |
| `WORKDIR /app` | Working directory | All subsequent commands run inside `/app` |
| `COPY package.json ./` | Copy file | Copy your dependency manifest into the image |
| `RUN npm install --omit=dev` | Execute command | Install production dependencies only (no dev tools) |
| `COPY server.js ./` | Copy file | Copy your application code |
| `EXPOSE 3000` | Document port | Tell Docker this service listens on 3000 (informational) |
| `CMD ["npm", "start"]` | Default command | When container starts, run `npm start` |

**Key insight**: Order matters! Docker caches layers:
- Rarely-changed files (base OS, dependencies) at the top
- Frequently-changed files (code) at the bottom
- This way, if you change only `server.js`, Docker reuses the cached Node/npm layers and rebuilds faster

**Compare with your frontend Dockerfile:**

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Why so simple? Nginx is a static file server. No dependencies to install, no build step. Just copy HTML and serve.

### 1.4 Docker Layers & Image Size

Every Dockerfile instruction creates a layer. Layers are stacked and cached:

```
Image: practice-backend:v5
├── Layer 1: FROM node:18-alpine (~160MB base)
├── Layer 2: WORKDIR /app (metadata only)
├── Layer 3: COPY package.json (KB)
├── Layer 4: RUN npm install (adds ~50MB for 67 packages)
├── Layer 5: COPY server.js (KB)
├── Layer 6: EXPOSE 3000 (metadata)
└── Layer 7: CMD ["npm", "start"] (metadata)
Total: ~210MB
```

**Optimization strategies:**
- Use minimal base images (`alpine` vs full Ubuntu)
- Combine RUN commands to reduce layers: `RUN apt-get update && apt-get install -y curl && apt-get clean`
- Put volatile files (code) last so earlier cached layers are reused

### 1.5 Image Building & Tagging

**Build command:**
```bash
minikube image build -t practice-backend:v5 ./backend
```

**Breakdown:**
- `minikube image build`: Build image using Minikube's Docker daemon (not your host Docker)
- `-t practice-backend:v5`: Tag the image with name and version
- `./backend`: Build context (directory containing Dockerfile)

**Tag format:** `<registry>/<repository>:<tag>`
- `practice-backend:v5` = no registry (local), repository=`practice-backend`, tag=`v5`
- Full example: `docker.io/myorg/myapp:1.2.3` (Docker Hub, myorg, tag 1.2.3)

**Why Minikube images instead of Docker Hub?**
- Faster (local, no network push/pull)
- `imagePullPolicy: Never` in your deployment tells Kubernetes to use cached local images
- Perfect for local dev, impractical for multi-machine production

---

## Part 2: Kubernetes Fundamentals

### 2.1 What is Kubernetes (K8s)?

Kubernetes is an **orchestration platform** for containerized applications. Think of it as an intelligent scheduler that:
- Deploys containers across machines (nodes)
- Ensures desired number of replicas stay running
- Routes traffic between components
- Manages storage, networking, secrets
- Scales up/down based on demand

**Why Kubernetes?**
- **High availability**: Auto-restarts failed containers
- **Scaling**: Easily run 1 or 1,000 replicas
- **Rolling updates**: Deploy new versions without downtime
- **Self-healing**: Replaces unhealthy pods
- **Cloud-native**: Standard across AWS, Azure, GCP, on-prem

### 2.2 Kubernetes Architecture: Cluster & Nodes

A **Kubernetes cluster** consists of:

```
Kubernetes Cluster (Minikube in your case)
├── Control Plane (Master)
│   ├── API Server (kubectl talks to this)
│   ├── Scheduler (decides which node to place pods)
│   ├── Controller Manager (ensures desired state)
│   └── etcd (key-value store of all cluster state)
└── Worker Nodes
    ├── Node 1 (minikube)
    │   ├── kubelet (agent, manages pods on this node)
    │   ├── container runtime (Docker, containerd, etc.)
    │   └── kube-proxy (handles networking)
    └── Pods running here
```

**Minikube**: Single machine running as both control plane and worker node. Perfect for learning, not production.

### 2.3 Pods: The Smallest Kubernetes Unit

A **Pod** is the smallest deployable unit in Kubernetes. It's a wrapper around one or more containers (usually one).

**Key characteristics:**
- Containers in a pod share a network namespace (same IP address, localhost)
- Can share storage volumes
- Tightly coupled, deployed together
- Ephemeral (created and destroyed)

Your cluster currently has two pods:

```bash
$ kubectl get pods
NAME                                    READY   STATUS
backend-deployment-c64f846f5-g6m4l     1/1     Running
frontend-deployment-686b8b7d4d-m578b   1/1     Running
```

Each pod runs one container (backend-deployment pod runs the backend container, etc.).

**Why not run containers directly?** Kubernetes doesn't manage containers directly—it manages pods. Pods add metadata, health checking, networking, storage management.

### 2.4 Deployments: Declarative Pod Management

A **Deployment** is a Kubernetes object that:
1. Defines a pod template (image, ports, env vars)
2. Specifies replica count (how many copies)
3. Manages updates, rollbacks, scaling

Your backend deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
spec:
  replicas: 1                    # Keep exactly 1 copy running
  selector:
    matchLabels:
      app: backend               # Manage pods with label "app: backend"
  template:
    metadata:
      labels:
        app: backend             # Pods created get this label
    spec:
      containers:
      - name: backend
        image: practice-backend:v5
        imagePullPolicy: Never   # Use cached local image
        ports:
        - containerPort: 3000
        readinessProbe:
          httpGet:
            path: /api/data
            port: 3000
          initialDelaySeconds: 5   # Wait 5s before first check
          periodSeconds: 10        # Check every 10s
        livenessProbe:
          httpGet:
            path: /api/data
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 20
        resources:
          requests:               # Minimum guaranteed
            cpu: 100m             # 0.1 cores
            memory: 128Mi
          limits:                 # Maximum allowed
            cpu: 250m
            memory: 256Mi
```

**What happens internally:**

1. **Desired state**: "I want 1 replica of the backend pod"
2. **Controller reconciliation loop**: Deployment controller constantly checks
   - "How many backend pods exist?" 
   - If count ≠ 1: Create or delete pods to match desired state
3. **Self-healing**: If a pod dies, Deployment creates a new one automatically

**Probes** (the health checks):
- **Readiness probe**: "Is this pod ready to receive traffic?" If fails, remove from load balancer
- **Liveness probe**: "Is this pod alive?" If fails, restart the container

Both probe your `/api/data` endpoint to verify the app is actually healthy, not just running.

**Resources:**
- **Requests**: Kubernetes reserves these resources for your pod. If node doesn't have 100m CPU, Kubernetes won't schedule your pod there.
- **Limits**: Pod can't exceed these. If it tries, Kubernetes kills it (OOMKilled for memory, throttled for CPU)

This prevents one greedy pod from starving others.

### 2.5 Services: Stable Network Endpoints

**Problem**: Pods are ephemeral. If a pod dies and a new one is created, it gets a new IP address. How do you reliably communicate with an ever-changing set of pods?

**Solution: Services**

A Service provides a stable IP and DNS name that load-balances traffic across matching pods.

Your backend service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP              # Internal only (default)
  selector:
    app: backend               # Load-balance to pods with "app: backend" label
  ports:
    - port: 3000              # Service listens on 3000
      targetPort: 3000         # Forward to port 3000 on pods
```

**How it works:**

```
Frontend Pod                    Kubernetes Network
    ↓
curl http://backend-service:3000/api/data
    ↓
Service (stable IP 10.105.245.89)  ← DNS resolves "backend-service" to this IP
    ↓
kube-proxy (iptables rules)
    ↓
Backend Pod 1 (10.244.0.17:3000)  ← Load-balanced target
```

**Service discovery**: Your frontend HTML uses `fetch('/api/data')` which becomes `http://backend-service:3000/api/data` through ingress routing, but if pods were calling each other directly, DNS would resolve `backend-service` automatically within the cluster.

**Service types:**

| Type | Purpose | Access |
|------|---------|--------|
| **ClusterIP** | Internal pod-to-pod communication | Inside cluster only |
| **NodePort** | Expose to host machine on a fixed port | `<nodeIP>:30001` |
| **LoadBalancer** | Cloud provider load balancer | External traffic, auto IP |
| **ExternalName** | Reference external service | DNS alias |

We changed from NodePort to ClusterIP because:
- Your frontend doesn't directly call backend via NodePort
- Instead, both go through Ingress (routing layer)
- ClusterIP is cleaner, more secure (no unnecessary ports exposed)

### 2.6 Ingress: HTTP(S) Routing

**Problem**: Services work, but they're Layer 4 (transport layer). How do you route HTTP requests based on URL paths or hostnames?

**Solution: Ingress**

An Ingress is a Kubernetes object that defines HTTP/HTTPS routing rules.

Your ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: practice-app-ingress
spec:
  ingressClassName: nginx        # Use nginx ingress controller
  rules:
  - http:
      paths:
      - path: /api               # Route /api/* to backend
        pathType: Prefix          # Match prefix, not exact
        backend:
          service:
            name: backend-service
            port:
              number: 3000
      - path: /                   # Route everything else to frontend
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

**Request flow:**

```
Browser: http://minikube-ip/api/data
    ↓
Ingress Controller (nginx) receives request
    ↓
Checks rules: path = "/api/data", matches "/api" rule
    ↓
Routes to backend-service:3000
    ↓
Service load-balances to backend pod
    ↓
Backend container serves /api/data endpoint
    ↓
Response: [{"id":1, "name":"Alice", ...}]
```

**Ingress Controller**: An Ingress object is just a declaration of intent. Something has to actually implement it. The nginx ingress controller is a pod in the cluster that:
1. Watches for Ingress objects
2. Generates nginx config based on rules
3. Routes traffic according to rules

Your Minikube has it enabled: `minikube addons enable ingress`

**Why Ingress instead of multiple LoadBalancer Services?**
- One public IP/hostname handles all HTTP routing
- Better resource efficiency (one LB vs many)
- Standard for cloud-native apps
- Supports hostname-based routing, TLS termination, path rewriting

### 2.7 DNS & Service Discovery

Inside the cluster, Kubernetes runs CoreDNS (a DNS server) that automatically resolves service names.

```
Pod trying to reach backend:
    ↓
DNS query: backend-service.default.svc.cluster.local
    ↓
CoreDNS resolves to service ClusterIP: 10.105.245.89
    ↓
Connection routed to that IP
    ↓
kube-proxy load-balances to backing pods
```

**Naming convention**: `<service>.<namespace>.svc.cluster.local`
- `<service>`: Your service name (backend-service)
- `<namespace>`: Kubernetes namespace, defaults to "default"
- `svc.cluster.local`: Suffix indicating it's a service in cluster.local domain

You can also shorthand within the same namespace: just `backend-service`

---

## Part 3: Your Application Architecture

### 3.1 Full Request Flow

Let's trace a user opening the app at `http://localhost:18080/`:

```
1. Browser (localhost:18080)
   └─→ kubectl port-forward exposes ingress controller:80 on localhost:18080
       └─→ Ingress Controller (nginx pod in ingress-nginx namespace)
           └─→ Matches path "/" → routes to frontend-service:80
               └─→ Service kube-proxy load-balances to frontend pod
                   └─→ Frontend Pod (nginx) serves index.html

2. Browser receives HTML, parses JavaScript:
   fetch('/api/data')
   └─→ Same host (localhost:18080)
   └─→ Browser sends: GET /api/data HTTP/1.1
       └─→ Ingress Controller receives request
           └─→ Matches path "/api" → routes to backend-service:3000
               └─→ Service load-balances to backend pod
                   └─→ Backend Pod (Node.js) runs app.get('/api/data')
                       └─→ Returns JSON array

3. Browser's JavaScript:
   .then(res => res.json())
   └─→ Parses response
   └─→ Dynamically creates HTML list of users
   └─→ Inserts into DOM
```

**Key observation**: 
- Same browser origin (localhost:18080)
- Ingress routes `/` to frontend, `/api` to backend
- Frontend and backend communicate through ingress, not directly
- Services provide stable DNS names within cluster

### 3.2 Container Lifecycle in Your App

When you run `kubectl apply -f app-deployment.yaml`:

```
1. API Server stores deployment in etcd
2. Deployment controller watches for changes
3. Controller creates ReplicaSet (manages pods for this deployment)
4. ReplicaSet controller creates Pods
5. Scheduler assigns Pod to a node (minikube)
6. kubelet on minikube notices new pod, pulls image
7. Container runtime (Docker) starts container
8. kubelet runs readinessProbe (HTTP GET /api/data)
   - If succeeds after initialDelaySeconds: Pod is Ready
   - Ingress starts routing traffic to it
9. kubelet runs livenessProbe periodically
   - If fails: Container is restarted
   - If succeeds: Pod stays alive
```

If you update the image tag (backend:v4 → v5):

```
1. kubectl applies new deployment
2. Deployment controller notices new spec
3. Deletes old ReplicaSet, creates new one
4. Creates new pod with v5 image
5. Only when new pod is Ready, terminates old pod
6. → Zero-downtime rolling update
```

---

## Part 4: Best Practices Implemented in Your App

### 4.1 Image Design

✅ **Multi-stage builds?** Not used (simple apps)
- For complex apps: build in one stage, copy artifacts to minimal final stage
- Reduces final image size significantly

✅ **Alpine Linux** for both images
- `node:18-alpine` (~160MB) vs `node:18` (~900MB)
- Has everything needed, minimal bloat

✅ **Production dependencies only**
- `npm install --omit=dev` excludes test frameworks, dev tools
- Reduces image size, attack surface, startup time

### 4.2 Deployment Design

✅ **Probes for health checking**
- Readiness: Ensures traffic only goes to healthy pods
- Liveness: Auto-restarts crashed containers
- Both use actual app endpoints, not just ping

✅ **Resource requests & limits**
- Requests ensure Kubernetes scheduler knows requirements
- Limits prevent runaway processes from crashing the node
- Enables fair resource sharing among pods

✅ **Replica count**
- Set to 1 for simplicity
- In production: set to 3+ for high availability
- Deployment automatically restarts dead pods

### 4.3 Service & Networking Design

✅ **ClusterIP for internal services**
- Pods communicate via stable DNS
- No unnecessary port exposure

✅ **Ingress for routing**
- Single entry point
- Path-based routing (not multiple external IPs)
- Standard for cloud-native apps

✅ **Namespace isolation** (implicit)
- Pods in default namespace
- Good security practice: separate namespaces for different apps/teams

### 4.4 Code Design

✅ **CORS headers** in backend
```javascript
res.header("Access-Control-Allow-Origin", "*");
```
- Allows frontend (different origin through ingress) to call backend
- In production: whitelist specific origins, not "*"

✅ **Relative URLs in frontend**
```javascript
fetch('/api/data')  // Not fetch('http://localhost:30001/api/data')
```
- Decouples from specific host/port
- Same code works locally, in cluster, anywhere

---

## Part 5: Common Kubernetes Patterns

### 5.1 ConfigMaps & Secrets

For configuration (not used here, but important):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  API_URL: "https://api.example.com"
  LOG_LEVEL: "debug"
```

Mount in pod:
```yaml
containers:
- name: app
  envFrom:
  - configMapRef:
      name: app-config
```

**Secrets**: Similar, but for sensitive data (passwords, tokens)
```yaml
kind: Secret
data:
  password: <base64-encoded>
```

### 5.2 StatefulSets vs Deployments

**Deployments** (your app): Stateless, interchangeable pods
- Order of startup doesn't matter
- Any pod can serve any request
- Perfect for web apps, APIs

**StatefulSets**: Stateful, ordered pods
- Each pod gets a unique identity (pod-0, pod-1)
- Persistent storage attached to each
- For databases, message queues, caches

Your app is stateless → Deployment is correct.

### 5.3 DaemonSets

Runs one pod per node. Used for:
- Log collectors (Fluentd)
- Monitoring agents (Prometheus)
- Network plugins
- Security tools

### 5.4 Jobs & CronJobs

**Jobs**: Run a task to completion, then stop
```yaml
kind: Job
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool:v1
      restartPolicy: Never
```

**CronJobs**: Run jobs on a schedule (like cron)
```yaml
kind: CronJob
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    ...
```

---

## Part 6: Troubleshooting Patterns

### 6.1 Pod won't start?

```bash
kubectl describe pod backend-deployment-c64f846f5-g6m4l
```
Check: Image pull errors, resource requests not available, readiness probe failing

### 6.2 Service not routing traffic?

```bash
kubectl get svc -o wide
kubectl get endpoints backend-service
```
Verify: Service has endpoints (matching pods), selectors are correct, pod labels match selector

### 6.3 Ingress not working?

```bash
kubectl get ingress
kubectl describe ingress practice-app-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```
Check: Ingress class is correct, service names/ports match, controller is running

### 6.4 Pod logs?

```bash
kubectl logs backend-deployment-c64f846f5-g6m4l
kubectl logs -f backend-deployment-c64f846f5-g6m4l  # Stream
```

### 6.5 Shell into pod for debugging?

```bash
kubectl exec -it backend-deployment-c64f846f5-g6m4l -- sh
```
Inside: `curl http://backend-service:3000/api/data` to test service discovery

---

## Part 7: Production Readiness Checklist

For your app to run in production:

- [ ] **Multiple replicas**: `replicas: 3` minimum for HA
- [ ] **Multi-region**: Run clusters in multiple regions, use global load balancer
- [ ] **Resource limits**: Carefully tune requests/limits based on actual usage
- [ ] **Monitoring**: Prometheus scraping pod metrics, Grafana dashboards
- [ ] **Logging**: Centralized logs (ELK, Stackdriver) instead of container logs
- [ ] **Secrets management**: Use Sealed Secrets or External Secrets, not ConfigMaps
- [ ] **Network policies**: Restrict pod-to-pod traffic (default: all pods can talk)
- [ ] **Pod security policies**: Prevent privileged containers
- [ ] **RBAC**: Fine-grained service account permissions
- [ ] **Backup**: Regular backups of etcd, persistent volumes
- [ ] **GitOps**: Use ArgoCD or Flux to manage manifests as code
- [ ] **TLS/mTLS**: Encrypt traffic, mutual auth between services

---

## Part 8: Quick Reference

### kubectl Commands

```bash
# View resources
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl describe pod <name>

# Deployment lifecycle
kubectl apply -f manifest.yaml      # Create/update
kubectl delete -f manifest.yaml     # Delete
kubectl rollout status deployment/backend-deployment  # Check rollout
kubectl rollout history deployment/backend-deployment # See past versions
kubectl rollout undo deployment/backend-deployment    # Revert to previous

# Debugging
kubectl logs <pod>
kubectl exec -it <pod> -- sh
kubectl port-forward svc/backend-service 3000:3000

# Scaling
kubectl scale deployment/backend-deployment --replicas=3

# Resource usage
kubectl top pods
kubectl top nodes
```

### Docker Commands (for reference)

```bash
docker build -t myapp:v1 .           # Build image
docker run -p 3000:3000 myapp:v1     # Run container
docker ps                             # List containers
docker exec -it <id> sh              # Shell into running container
docker logs <id>                      # View container logs
```

---

## Conclusion

Your application demonstrates:
1. **Docker**: Containerizing Node.js backend and static frontend
2. **Kubernetes**: Deploying, scaling, health checking, networking
3. **Modern architecture**: Services for discovery, Ingress for routing
4. **DevOps best practices**: Health probes, resource management, image optimization

The journey:
- Code → Docker image (encapsulation)
- Image → Container in pod (runtime)
- Pods → Service (networking)
- Service → Ingress (external routing)
- All orchestrated by Kubernetes (automation at scale)

This pattern applies to microservices apps with hundreds of services—same principles, just more complex routing and many more pods.
