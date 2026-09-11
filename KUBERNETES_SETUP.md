# Cosmos App - Kubernetes & CI/CD Complete Setup Guide

## 🎯 Overview

This guide covers the complete setup of **Cosmos App** on Kubernetes with:

- ✅ GitHub Actions CI/CD (auto-build & push Docker images)
- ✅ Kubernetes Deployments (API + Frontend)
- ✅ Horizontal Pod Autoscaling (HPA)
- ✅ Persistent Storage (PersistentVolumes)
- ✅ Health Checks & Monitoring

**Status**: Production-ready for local K8s (Docker Desktop) and cloud deployments (EKS, AKS, GKE).

---

## 📋 Prerequisites

- Docker Desktop with Kubernetes enabled (v1.27+)
- `kubectl` CLI installed
- Git & GitHub account
- Docker Hub account (for image registry)

**Verify setup:**

```powershell
docker --version
kubectl version --client
git --version
```

---

## Part 1: GitHub Actions CI/CD Setup

### 1.1 Add Docker Hub Credentials to GitHub Secrets

1. Go to: <https://github.com/JefeEyuel-art-prod/Cosmos-NMV/settings/secrets/actions>
2. Click **New repository secret** → Add these two:

**Secret 1:**

```
Name: DOCKER_USERNAME
Value: eyuel21
```

**Secret 2:**

```
Name: DOCKER_PASSWORD
Value: [Your Docker Hub Personal Access Token]
```

**How to get Docker Hub token:**

1. Go to: <https://hub.docker.com/settings/security>
2. Click **New Access Token**
3. Give it a name: `github-actions`
4. Select scope: `Read, Write & Delete`
5. Copy the token and paste in GitHub secret

### 1.2 CI/CD Workflow Overview

**Location:** `.github/workflows/build.yml`

**Triggers on:**

- Push to `main` or `develop` branches
- Changes to `cosmos-app/` or `backend/` directories

**Jobs:**

1. **build-api**: Builds Node.js backend image
2. **build-frontend**: Builds Vite frontend image (nginx)
3. **deploy-k8s**: (Optional) Updates K8s deployments with new images

**Image tags created:**

- `eyuel21/cosmos-api:latest` (always latest build)
- `eyuel21/cosmos-api:<git-sha>` (version-specific tag)
- Same for `cosmos-frontend`

**Test it:**

```powershell
# Make a commit to main
git add .
git commit -m "Test CI/CD trigger"
git push

# Watch workflow run
# Go to: https://github.com/JefeEyuel-art-prod/Cosmos-NMV/actions
```

---

## Part 2: Kubernetes Deployment

### 2.1 Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                       │
├──────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────────┐          ┌──────────────────┐      │
│  │   API Service    │          │ Frontend Service │      │
│  │   (NodePort 5000)│          │ (NodePort 5173)  │      │
│  └────────┬─────────┘          └────────┬─────────┘      │
│           │                             │                 │
│  ┌────────▼──────────┐        ┌────────▼─────────────┐   │
│  │  API Deployment   │        │ Frontend Deployment  │   │
│  │  (2 replicas)     │        │  (2 replicas)        │   │
│  └────────┬──────────┘        └────────┬─────────────┘   │
│           │                             │                 │
│  ┌────────▼──────────────────────────────▼────────┐      │
│  │         HPA (Auto-scaling)                     │      │
│  │  - API: 2-5 replicas (70% CPU, 80% memory)   │      │
│  │  - Frontend: 2-4 replicas (75% CPU, 85%)     │      │
│  └─────────────────────────────────────────────────┘     │
│           │                             │                 │
│  ┌────────▼──────────┐        ┌────────▼─────────────┐   │
│  │  PersistentVolume │        │ PersistentVolume     │   │
│  │  (API data: 5Gi)  │        │ (Frontend: 2Gi)      │   │
│  └───────────────────┘        └──────────────────────┘   │
│                                                            │
└──────────────────────────────────────────────────────────┘
```

### 2.2 Deploy to Kubernetes

**Step 1: Apply Storage (PersistentVolumes & Claims)**

```powershell
kubectl apply -f k8s-storage.yaml

# Verify
kubectl get pv
kubectl get pvc
```

**Step 2: Apply API Deployment**

```powershell
kubectl apply -f k8s-api-deployment.yaml

# Verify
kubectl get deployment cosmos-api
kubectl get pods -l app=cosmos-api
kubectl logs -f deployment/cosmos-api
```

**Step 3: Apply Frontend Deployment**

```powershell
kubectl apply -f k8s-frontend-deployment.yaml

# Verify
kubectl get deployment cosmos-frontend
kubectl get pods -l app=cosmos-frontend
kubectl logs -f deployment/cosmos-frontend
```

**Step 4: Apply Horizontal Pod Autoscaler (HPA)**

```powershell
kubectl apply -f k8s-hpa.yaml

# Verify (metrics may show <unknown> for 1-2 minutes)
kubectl get hpa
kubectl describe hpa cosmos-api-hpa
```

**Step 5: Apply Ingress (Optional - for external traffic)**

```powershell
kubectl apply -f k8s-ingress.yaml

# Verify
kubectl get ingress
kubectl describe ingress cosmos-ingress
```

---

## Part 3: Verification & Testing

### 3.1 Check Deployment Status

```powershell
# All pods running
kubectl get pods
# Expected output:
# cosmos-api-xxx              1/1 Running
# cosmos-frontend-xxx         1/1 Running
# (2 replicas of each)

# Deployments ready
kubectl get deployments

# Services created
kubectl get svc

# HPA status
kubectl get hpa
```

### 3.2 Test API Endpoint

```powershell
# Port-forward API service
kubectl port-forward svc/cosmos-api 5000:5000 &

# Wait 2 seconds
Start-Sleep -Seconds 2

# Test health endpoint
Invoke-WebRequest http://localhost:5000/api/health -UseBasicParsing

# Expected response: 200 OK + JSON body with status
```

### 3.3 Test Frontend

```powershell
# Port-forward Frontend service
kubectl port-forward svc/cosmos-frontend 5173:5173 &

# Wait 2 seconds
Start-Sleep -Seconds 2

# Test frontend
Invoke-WebRequest http://localhost:5173 -UseBasicParsing

# Expected response: 200 OK + HTML body (cosmos-app)
```

### 3.4 Monitor Autoscaling

```powershell
# Watch HPA in real-time
kubectl get hpa -w

# In another terminal, generate load on API
for ($i = 0; $i -lt 100; $i++) {
  Invoke-WebRequest http://localhost:5000/api/health -UseBasicParsing | Out-Null
  Write-Host "Request $i"
}

# Watch replicas scale from 1 to 5 as CPU spikes above 70%
```

---

## Part 4: Manifest Files Reference

### 4.1 k8s-api-deployment.yaml

**Contains:**

- `ConfigMap`: Environment variables (NODE_ENV, PORT, etc.)
- `Deployment`: 1 replica (can scale to 5 via HPA)
  - Image: `eyuel21/cosmos-api:latest`
  - Port: 5000
  - Resources: 256Mi memory / 100m CPU (requests), 512Mi / 500m (limits)
  - Liveness probe: HTTP GET /api/health (30s delay)
  - Volume mount: `/app/data` (PVC)
- `Service`: ClusterIP exposing port 5000

### 4.2 k8s-frontend-deployment.yaml

**Contains:**

- `ConfigMap`: VITE_API_URL pointing to API service
- `Deployment`: 1 replica (can scale to 4 via HPA)
  - Image: `eyuel21/cosmos-frontend:latest` (Nginx)
  - Port: 5173
  - Resources: Same as API
  - Liveness probe: TCP socket on port 5173 (30s delay)
  - Volume mount: `/var/cache/nginx` (PVC)
- `Service`: ClusterIP exposing port 5173

### 4.3 k8s-storage.yaml

**Contains:**

- `StorageClass`: `cosmos-storage` (no-provisioner for local testing)
- `PersistentVolume`: `cosmos-api-pv` (5Gi, hostPath: `/data/cosmos-api`)
- `PersistentVolumeClaim`: `cosmos-api-pvc` (requests 5Gi)
- `PersistentVolume`: `cosmos-frontend-pv` (2Gi, hostPath: `/data/cosmos-frontend`)
- `PersistentVolumeClaim`: `cosmos-frontend-pvc` (requests 2Gi)

**Note:** `hostPath` is for local testing only. For production:

- **AWS**: Use EBS volumes (change provisioner to `ebs.csi.aws.com`)
- **Azure**: Use Azure Disks (change provisioner to `disk.csi.azure.com`)
- **GCP**: Use Persistent Disks (change provisioner to `pd.csi.storage.gke.io`)

### 4.4 k8s-hpa.yaml

**Contains:**

- `HorizontalPodAutoscaler` for `cosmos-api`:
  - Min: 2 replicas, Max: 5 replicas
  - Targets: 70% CPU, 80% memory utilization
  - Scale-up: Instant (0s stabilization)
  - Scale-down: After 5 minutes of low usage (300s)

- `HorizontalPodAutoscaler` for `cosmos-frontend`:
  - Min: 2 replicas, Max: 4 replicas
  - Targets: 75% CPU, 85% memory utilization
  - Same scaling behavior as API

### 4.5 k8s-ingress.yaml

**Contains:**

- `Ingress` routing rules:
  - `/api/*` → `cosmos-api:5000`
  - `/` → `cosmos-frontend:5173`
  - Host: `cosmos.local` (change to your domain)

**Note:** Requires nginx-ingress-controller. Install with:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

---

## Part 5: Troubleshooting

### Issue: Pod stuck in `CrashLoopBackOff`

**Cause:** Application crashes or liveness probe fails too quickly

**Solution:**

```powershell
# Check logs
kubectl logs deployment/cosmos-frontend

# Increase liveness probe delay
kubectl edit deployment cosmos-frontend
# Change: initialDelaySeconds: 30 (was 15)

# Check pod events
kubectl describe pod <pod-name>
```

### Issue: HPA shows `<unknown>` for metrics

**Cause:** Metrics Server not installed

**Solution:**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait 1-2 minutes, then check
kubectl get hpa
```

### Issue: PersistentVolume not binding

**Cause:** PVC and PV mismatch (storage class, access mode, size)

**Solution:**

```powershell
# Check PV and PVC status
kubectl get pv
kubectl get pvc

# Describe to see errors
kubectl describe pvc cosmos-api-pvc
kubectl describe pv cosmos-api-pv

# For hostPath, ensure host directory exists
mkdir -p C:\data\cosmos-api
mkdir -p C:\data\cosmos-frontend
```

### Issue: Image pull errors

**Cause:** Image doesn't exist on Docker Hub or auth failed

**Solution:**

```powershell
# Verify image exists
docker pull eyuel21/cosmos-api:latest
docker pull eyuel21/cosmos-frontend:latest

# If pull fails, rebuild and push
docker build -t eyuel21/cosmos-api:latest ./backend
docker push eyuel21/cosmos-api:latest

# Then re-apply deployment
kubectl rollout restart deployment/cosmos-api
```

### Issue: Service not accessible

**Cause:** Port-forward not active or service not found

**Solution:**

```powershell
# List services
kubectl get svc

# Check service endpoints
kubectl describe svc cosmos-api

# Port-forward again
kubectl port-forward svc/cosmos-api 5000:5000 &
```

---

## Part 6: Production Checklist

- [ ] **Domain & TLS**: Update Ingress with your domain + SSL certificate
- [ ] **Resource Quotas**: Set namespace limits (CPU, memory, pod count)
- [ ] **Network Policies**: Restrict traffic between pods
- [ ] **RBAC**: Create service accounts with minimal permissions
- [ ] **Secrets Management**: Use sealed-secrets or vault (not ConfigMap)
- [ ] **Database**: Add PostgreSQL/MongoDB StatefulSet with persistent storage
- [ ] **Monitoring**: Install Prometheus + Grafana
- [ ] **Logging**: Add ELK Stack or Loki
- [ ] **Backup**: Enable etcd backups for cluster data
- [ ] **CI/CD**: Connect GitHub Actions to auto-deploy on push
- [ ] **Load Testing**: Verify HPA scales correctly under load
- [ ] **Disaster Recovery**: Document recovery procedures

---

## Part 7: Advanced Topics

### 7.1 Multi-Environment Deployment

**Directory structure:**

```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── k8s-api-deployment.yaml
│   ├── k8s-frontend-deployment.yaml
│   └── ...
├── overlays/
│   ├── development/
│   │   ├── kustomization.yaml
│   │   └── patch-replicas.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       ├── kustomization.yaml
│       └── patch-resources.yaml
```

**Deploy with Kustomize:**

```bash
kubectl apply -k k8s/overlays/production
```

### 7.2 Helm Chart

**Create Helm chart for reusability:**

```bash
helm create cosmos-app
# Edit values.yaml with image, replicas, etc.
# Deploy anywhere: helm install cosmos-app ./cosmos-app
```

### 7.3 GitOps with ArgoCD

**Automatic sync from Git:**

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# ArgoCD watches your repo and auto-applies changes
```

---

## 📚 Useful Commands Cheat Sheet

```powershell
# Deployments
kubectl get deployments
kubectl describe deployment cosmos-api
kubectl rollout status deployment/cosmos-api
kubectl rollout restart deployment/cosmos-api

# Pods
kubectl get pods
kubectl logs pod <pod-name>
kubectl logs -f deployment/cosmos-api  # follow logs
kubectl exec -it <pod-name> -- /bin/sh  # shell access
kubectl describe pod <pod-name>

# Services
kubectl get svc
kubectl port-forward svc/cosmos-api 5000:5000

# Storage
kubectl get pv
kubectl get pvc
kubectl describe pvc cosmos-api-pvc

# HPA
kubectl get hpa
kubectl describe hpa cosmos-api-hpa
kubectl top nodes
kubectl top pods

# Troubleshooting
kubectl get events
kubectl explain deployment
kubectl api-resources
```

---

## 📞 Support & Resources

- **Kubernetes Docs**: <https://kubernetes.io/docs/>
- **Docker Documentation**: <https://docs.docker.com/>
- **GitHub Actions**: <https://docs.github.com/en/actions>
- **kubectl Cheat Sheet**: <https://kubernetes.io/docs/reference/kubectl/cheatsheet/>

---

**Status**: ✅ Production-ready

**Last Updated**: September 2026

**Maintained by**: Cosmos App Team
