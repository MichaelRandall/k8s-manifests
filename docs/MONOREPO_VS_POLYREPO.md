# Monorepo vs. Polyrepo: Complete Comparison

This guide shows the real-world differences between the two architectures you've explored.

## Side-by-Side Comparison

### Repository Structure

**Monorepo (k8s-users repo)**
```
github.com/MichaelRandall/k8s-users
├── backend/
│   ├── Dockerfile
│   ├── server.js
│   └── package.json
├── frontend/
│   ├── Dockerfile
│   └── index.html
├── .github/workflows/
│   └── build-and-push.yml     ← ONE workflow builds BOTH
├── app-deployment.yaml
└── k8s-manifests/base/app-ingress.yaml
```

**Polyrepo (Two separate repos)**
```
github.com/MichaelRandall/k8s-back
├── Dockerfile
├── server.js
├── package.json
└── .github/workflows/
    └── build-and-push.yml     ← BACKEND workflow

github.com/MichaelRandall/k8s-front
├── Dockerfile
├── index.html
└── .github/workflows/
    └── build-and-push.yml     ← FRONTEND workflow

github.com/MichaelRandall/k8s-users  (or k8s-manifests)
├── app-deployment.yaml
└── k8s-manifests/base/app-ingress.yaml
```

---

## Detailed Comparison Table

| Aspect | Monorepo | Polyrepo |
|--------|----------|----------|
| **GitHub Repos** | 1 | 2-3 (backend, frontend, manifests) |
| **Workflows** | 1 (builds both) | 2 (one per service) |
| **Docker Hub repos** | 2 (k8s-back, k8s-front) | 2 (same) |
| **Image versioning** | Synchronized (backend:v5 + frontend:v2 released together) | Independent (backend:v5, frontend:v3 at different times) |
| **Code commit** | One `git push` | Two separate `git push` commands |
| **CI/CD trigger** | One workflow run | Two independent workflow runs |
| **Kubernetes manifest** | Same `app-deployment.yaml` | Same `k8s-manifests/base/app-deployment-polyrepo.yaml` (but pulls from independent images) |
| **Rollback** | Both services rollback together | Roll back one service without affecting other |
| **Team structure** | Shared ownership | Separate teams (Backend Team, Frontend Team) |
| **Release process** | Single release (app v1.0.0 = backend v1.0 + frontend v1.0) | Decoupled (backend v2.0 released while frontend still v1.0) |
| **Coordination** | Easier (one change per commit) | Harder (need to track which version of each service is deployed) |

---

## Development Workflow Comparison

### Monorepo: Both Developers

```bash
# Developer 1 works on backend
cd k8s-users/backend
nano server.js
git add server.js
git commit -m "Add endpoint"
git push origin main
# Triggers: workflow builds k8s-back:latest AND k8s-front:latest
#           Both images rebuild even if frontend unchanged

# Developer 2 works on frontend (same time)
cd k8s-users/frontend
nano index.html
git add index.html
git commit -m "Update UI"
git push origin main
# Triggers: workflow builds k8s-back:latest AND k8s-front:latest
#           Both images rebuild again
```

**Result**: Both images in Docker Hub are new, both are "in sync"

### Polyrepo: Separate Teams

```bash
# Backend Team works independently
cd k8s-back
nano server.js
git add server.js
git commit -m "Add endpoint"
git push origin main
# Triggers: ONLY backend workflow
# Result: k8s-back:latest updated on Docker Hub
#         k8s-front:latest unchanged

# Frontend Team works independently (same time)
cd k8s-front
nano index.html
git add index.html
git commit -m "Update UI"
git push origin main
# Triggers: ONLY frontend workflow
# Result: k8s-front:latest updated on Docker Hub
#         k8s-back:latest unchanged
```

**Result**: Services deploy independently, can test and release at different schedules

---

## Versioning Implications

### Monorepo Release

You follow **semantic versioning** as a single unit:

```bash
# Release v2.0.0
git tag v2.0.0
git push origin v2.0.0

# GitHub Actions creates:
# - mran66/k8s-back:2.0.0
# - mran66/k8s-front:2.0.0

# Kubernetes deployment:
# backend: mran66/k8s-back:2.0.0
# frontend: mran66/k8s-front:2.0.0
# Both at the same version, released together
```

### Polyrepo Release

Services version independently:

```bash
# Backend Team releases v2.0.0
cd k8s-back
git tag v2.0.0
git push origin v2.0.0
# GitHub Actions creates: mran66/k8s-back:2.0.0

# Frontend Team still on v1.0.0 (no changes)
# OR Frontend releases v2.5.0 (different versioning)
# GitHub Actions creates: mran66/k8s-front:2.5.0

# Kubernetes deployment can mix versions:
# backend: mran66/k8s-back:2.0.0
# frontend: mran66/k8s-front:1.0.0
# Deployed at different times, different versions
```

---

## API Compatibility Scenarios

### Breaking Change in Polyrepo

Frontend expects: `GET /api/data` returns `[{id, name, role, status}]`

Backend Team makes breaking change: `GET /api/data` returns `[{userId, displayName, department}]`

**Monorepo**: 
- Both changes deployed together ✅
- Tested together before release ✅
- No compatibility issues

**Polyrepo**:
- Backend deploys v2.0.0 with new API ❌
- Frontend still expects old API ❌
- App breaks until Frontend Team updates and redeploys ❌

**Solutions**:
1. **API versioning**: Backend supports both `/api/v1/data` and `/api/v2/data`
2. **Backward compatibility**: Backend changes are additive, never breaking
3. **Coordination**: Frontend Team notified before Backend Team releases
4. **API contracts**: Use OpenAPI/Swagger to document expectations

---

## When to Use Each

### Use **Monorepo** if:

✅ Small team (1-5 people)  
✅ Frontend and backend are tightly coupled  
✅ Both need to release together  
✅ Single service (not microservices)  
✅ Learning/prototyping  
✅ Simpler to manage initially  

### Use **Polyrepo** if:

✅ Large team (10+ people)  
✅ Separate teams own frontend and backend  
✅ Services release independently  
✅ Multiple microservices (5+)  
✅ Different technology stacks per service  
✅ Services have different SLAs/uptime requirements  
✅ Teams want to move at different velocities  

### Real-World Examples

**Monorepo:**
- Google (Bazel-managed monorepo, 40M+ lines)
- Meta (buck-managed monorepo)
- Stripe (one primary repo for some services)

**Polyrepo:**
- Netflix (100s of microservices)
- Uber (thousands of repos)
- AWS (each service separate)
- Most SaaS companies

---

## Migration Path

### Monorepo → Polyrepo

If you start with monorepo and later split:

```
Week 1: Monorepo works fine
Week 4: Team grows to 3 people, coordination harder
Week 8: Decide to split services
        Create k8s-back and k8s-front repos
        Copy code over
        Add workflows to each
        Test deployment
Week 9: Both teams work independently
```

### Polyrepo → Monorepo

If polyrepo gets too complex:

```
Quarter 2: Polyrepo workflow works but complex
Quarter 3: Too many repos to manage (15+)
Quarter 4: Decide to merge back to monorepo
           Or use Monorepo tool (Nx, Lerna)
```

---

## Hybrid: Monorepo with Multiple Services

As you grow, you might use a **hybrid approach**:

```
github.com/MichaelRandall/k8s-services  (monorepo)
├── services/
│   ├── backend/
│   │   ├── Dockerfile
│   │   └── server.js
│   ├── frontend/
│   │   ├── Dockerfile
│   │   └── index.html
│   ├── payment-service/
│   │   ├── Dockerfile
│   │   └── service.go
│   └── analytics-service/
│       ├── Dockerfile
│       └── service.py
├── .github/workflows/
│   ├── build-backend.yml      # Builds ONLY if backend/ changed
│   ├── build-frontend.yml     # Builds ONLY if frontend/ changed
│   ├── build-payment.yml      # Builds ONLY if payment-service/ changed
│   └── build-analytics.yml    # Builds ONLY if analytics-service/ changed
└── k8s-manifests/
    └── deployments.yaml
```

This gives you:
- **Monorepo benefits**: Single clone, shared CI/CD infrastructure
- **Polyrepo benefits**: Independent builds, independent releases

Tools like **Nx**, **Lerna**, or **Bazel** enable this pattern.

---

## Your Decision: Monorepo vs. Polyrepo?

For your learning project:

| Stage | Recommendation |
|-------|-----------------|
| **Now** | **Monorepo** (k8s-users repo as-is) — simpler, works well |
| **If team grows to 2-3** | **Consider polyrepo** — what you're learning now |
| **If 10+ services** | **Hybrid monorepo** — multiple services, smart CI/CD |

My suggestion: **Start with polyrepo structure now** since you explicitly want to learn about independent teams and releases. You'll understand both patterns, and it's easier to merge back to monorepo than to split from it.

---

## Implementation Quick Reference

- **Monorepo setup**: See `GITHUB_ACTIONS_SETUP.md` + use current `k8s-users` repo
- **Polyrepo setup**: See `POLYREPO_SETUP.md` + create `k8s-back` and `k8s-front` repos
- **Kubernetes manifests**: 
  - Monorepo: `k8s-manifests/overlays/dockerhub/app-deployment-dockerhub.yaml`
  - Polyrepo: `k8s-manifests/base/app-deployment-polyrepo.yaml` (identical, just pulls from independent images)

Both are equivalent to Kubernetes—it doesn't care how images were built, only that they exist on Docker Hub.
