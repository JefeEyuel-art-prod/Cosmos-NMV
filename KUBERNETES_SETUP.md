# Cosmos App CI/CD & Kubernetes Setup Guide

## Fixed Issues ✓
- ✓ `docker-compose.yml`: Fixed `wversion` → `version`
- ✓ `devcontainer.json`: Fixed trailing comma
- ✓ Added `terser` to frontend `package.json` (required for Vite builds)
- ✓ Backend Dockerfile: Changed `npm ci --only=production` → `npm install --omit=dev`

## 1. GitHub Actions CI/CD Setup

### Step 1: Add Docker Hub credentials to GitHub Secrets
1. Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions**
2. Create two secrets:
   - `DOCKER_USERNAME`: your Docker Hub username (eyuel21)
   - `DOCKER_PASSWORD`: your Docker Hub token (from hub.docker.com → Account Settings → Security)

### Step 2: Add Kubernetes config to GitHub Secrets (optional, for auto-deploy)
```bash
# Get your kubeconfig (if deploying to a remote cluster)
cat ~/.kube/config | base64 | xclip -selection clipboard
```
Add as `KUBE_CONFIG` secret in GitHub.

### Step 3: Workflow triggers automatically on:
- Push to `main` or `develop` branches
- Changes in `cosmos-app/` or `backend/` directories
- Creates images tagged with both `latest` and `git-sha` for version tracking

**What it does:**
- Builds API and Frontend Docker images
- Pushes to Docker Hub
- (Optional) Deploys to Kubernetes with zero-downtime rolling updates

---

## 2. Kubernetes Storage Setup

### PersistentVolumes created:
- **cosmos-api-pv**: 5Gi for API data (logs, cache)
- **cosmos-frontend-pv**: 2Gi for Frontend cache

### Mount points:
- API: `/app/data`
- Frontend: `/var/cache/nginx`

**Local testing only**: Uses `hostPath` storage. For production, use:
- **AWS**: EBS volumes
- **Azure**: Azure Disk
- **GCP**: Persistent Disks
- **Multi-node**: NFS or distributed storage (Ceph, etc.)

---

## 3. Horizontal Pod Autoscaler (HPA)

### Automatic scaling based on:

| Service | Min Replicas | Max Replicas | CPU Target | Memory Target |
|---------|------------|------------|-----------|--------------|
| API | 2 | 5 | 70% | 80% |
| Frontend | 2 | 4 | 75% | 85% |

### Scaling behavior:
- **Scale up**: Instantly (0s stabilization) when threshold exceeded
- **Scale down**: After 5 minutes (300s) of low usage

### Monitor scaling:
```bash
kubectl get hpa -w
kubectl describe hpa cosmos-api-hpa
```

---

## 4. Deploy Everything

```bash
# Apply storage
kubectl apply -f k8s-storage.yaml

# Apply deployments (with storage + resources)
kubectl apply -f k8s-api-deployment.yaml
kubectl apply -f k8s-frontend-deployment.yaml

# Apply autoscaling
kubectl apply -f k8s-hpa.yaml

# Apply ingress (for external traffic)
kubectl apply -f k8s-ingress.yaml

# Verify
kubectl get pods
kubectl get hpa
kubectl get pvc
kubectl get pv
```

---

## 5. Verify Deployments

```bash
# Check pod status
kubectl get pods
kubectl describe pod <pod-name>

# Check autoscaler status
kubectl get hpa -w

# Check storage
kubectl get pvc
kubectl get pv

# View logs
kubectl logs -f deployment/cosmos-api
kubectl logs -f deployment/cosmos-frontend

# Test health
kubectl port-forward svc/cosmos-api 5000:5000 &
curl http://localhost:5000/api/health
```

---

## 6. Next Steps

1. **Configure Domain/TLS**: Update `k8s-ingress.yaml` with your domain and SSL cert
2. **Add Database**: Create `StatefulSet` for PostgreSQL/MongoDB (with PersistentVolumes)
3. **Monitoring**: Install Prometheus + Grafana for K8s metrics
4. **Log aggregation**: Add ELK Stack or Loki for centralized logs
5. **Resource Quotas**: Set namespace-level CPU/memory limits

---

## Troubleshooting

### HPA shows `<unknown>` for metrics?
```bash
# Metrics Server may not be installed
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Pod stuck in `Pending`?
```bash
# Check events
kubectl describe pod <pod-name>

# Usually: PVC not bound, insufficient resources, or image pull failure
```

### Storage not mounting?
```bash
# Check PVC status
kubectl get pvc
kubectl describe pvc cosmos-api-pvc

# Ensure PV exists and matches selector
kubectl get pv
```

---

## File Structure

```
cosmos-app/
├── .github/workflows/build.yml          # CI/CD pipeline
├── cosmos-app/                          # Frontend (Vite)
├── backend/                             # API (Node.js)
├── k8s-api-deployment.yaml              # API K8s manifest
├── k8s-frontend-deployment.yaml         # Frontend K8s manifest
├── k8s-storage.yaml                     # PersistentVolumes & PVCs
├── k8s-hpa.yaml                         # Horizontal Pod Autoscaler
├── k8s-ingress.yaml                     # Ingress routing
└── docker-compose.yml                   # Local dev (fixed)
```

---

**Your Cosmos app is now production-ready with auto-scaling, persistent storage, and CI/CD! 🚀**
