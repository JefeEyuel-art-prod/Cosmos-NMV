# CI/CD Pipeline Setup Guide

## Overview

This CI/CD pipeline automatically:
- ✅ Tests code on every push
- ✅ Builds and pushes Docker images
- ✅ Scans images for vulnerabilities (Trivy)
- ✅ Deploys to staging on `develop` branch
- ✅ Deploys to production on `main` branch
- ✅ Sends Slack notifications

**Workflow:**
```
Push Code → Test → Build Images → Scan → Deploy (Staging or Production) → Notify
```

---

## Part 1: Set Up GitHub Secrets

### 1.1 Docker Registry Credentials

Go to: https://github.com/JefeEyuel-art-prod/Cosmos-NMV/settings/secrets/actions

Create these secrets:

**Secret 1: `DOCKER_USERNAME`**
```
Value: eyuel21
```

**Secret 2: `DOCKER_PASSWORD`**
```
Value: [Your Docker Hub Personal Access Token]
```

### 1.2 Kubernetes Configuration

**Get your kubeconfig:**

**For local Docker Desktop K8s:**
```powershell
$kubeconfig = [System.IO.File]::ReadAllText("$env:USERPROFILE\.kube\config")
$encoded = [System.Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($kubeconfig))
Write-Host $encoded
```

**For cloud K8s (AWS EKS, Azure AKS, GCP GKE):**
```bash
# AWS EKS
aws eks update-kubeconfig --region us-east-1 --name cosmos-cluster

# Azure AKS
az aks get-credentials --resource-group myResourceGroup --name cosmosCluster

# GCP GKE
gcloud container clusters get-credentials cosmos-cluster --zone us-central1-a

# Get base64 encoded config
cat ~/.kube/config | base64 | tr -d '\n'
```

**Create GitHub secrets:**

**Secret 3: `KUBE_CONFIG_STAGING`** (optional)
```
Value: [base64-encoded kubeconfig for staging cluster]
```

**Secret 4: `KUBE_CONFIG_PROD`**
```
Value: [base64-encoded kubeconfig for production cluster]
```

### 1.3 Slack Notifications (Optional)

**Get Slack webhook:**

1. Go to https://api.slack.com/apps
2. Create New App → From scratch
3. Name: "Cosmos Deployments"
4. Select your workspace
5. Incoming Webhooks → Activate
6. Add New Webhook to Workspace
7. Select channel: #deployments
8. Copy Webhook URL

**Create GitHub secret:**

**Secret 5: `SLACK_WEBHOOK`**
```
Value: https://hooks.slack.com/services/T.../B.../...
```

---

## Part 2: Set Up GitHub Environments

Go to: https://github.com/JefeEyuel-art-prod/Cosmos-NMV/settings/environments

### 2.1 Create Staging Environment

1. Click **New environment**
2. Name: `staging`
3. Add deployment branches: `develop`
4. Add secrets:
   - `KUBE_CONFIG_STAGING` (if separate from prod)

### 2.2 Create Production Environment

1. Click **New environment**
2. Name: `production`
3. Add deployment branches: `main`
4. **Enable required reviewers**: Check "Required reviewers"
   - Add yourself or team members
   - This requires approval before prod deployment
5. Add secrets:
   - `KUBE_CONFIG_PROD`

---

## Part 3: Create Kubernetes Namespaces

Your pipeline deploys to `staging` and `production` namespaces. Create them:

```powershell
# Create namespaces
kubectl create namespace staging
kubectl create namespace production

# Label for Istio injection (optional)
kubectl label namespace staging istio-injection=enabled
kubectl label namespace production istio-injection=enabled

# Verify
kubectl get namespaces
```

---

## Part 4: Deploy K8s Manifests to Each Environment

**For Staging:**
```powershell
kubectl apply -f k8s-microservices.yaml -n staging
kubectl apply -f k8s-istio-gateway.yaml -n staging
```

**For Production:**
```powershell
kubectl apply -f k8s-microservices.yaml -n production
kubectl apply -f k8s-istio-gateway.yaml -n production
```

---

## Part 5: Test the Pipeline

### 5.1 Trigger staging deployment

```powershell
cd cosmos-app
git checkout develop

# Make a small change
echo "# Test deploy" >> README.md

git add .
git commit -m "Test staging deployment"
git push origin develop
```

Watch: https://github.com/JefeEyuel-art-prod/Cosmos-NMV/actions

**Expected output:**
```
✅ test-api - PASSED
✅ test-frontend - PASSED
✅ build-images - Docker images built & pushed
✅ Trivy scan - No critical vulnerabilities
✅ deploy-staging - Deployed to staging namespace
🔔 Slack notification sent
```

### 5.2 Trigger production deployment

```powershell
git checkout main
git merge develop  # or create a PR

git push origin main
```

**Expected output:**
```
✅ All tests pass
✅ Images built & pushed
✅ Requires approval (if reviewers enabled)
✅ Once approved: deploy-production runs
✅ Health checks pass
🔔 Slack notification sent
```

---

## Part 6: Pipeline Stages Explained

### **Stage 1: Test**
- Installs dependencies
- Runs linting (`npm run lint`)
- Builds code (`npm run build`)
- Runs tests (`npm test`)

### **Stage 2: Build & Push Images**
- Tags images: `latest`, `develop/main`, `sha`
- Pushes to Docker Hub
- Scans with Trivy for vulnerabilities

### **Stage 3: Deploy (Staging or Production)**
- Uses `kubectl set image` to update deployments
- Waits for rollout completion (5-10 min timeout)
- Runs smoke tests (health checks)
- Sends Slack notification

### **Stage 4: Notifications**
- **Success**: ✅ Green message with commit/author
- **Failure**: ❌ Red alert with debug link

---

## Part 7: Manual Rollback

If something breaks in production:

```powershell
# View rollout history
kubectl rollout history deployment/cosmos-api -n production

# Rollback to previous version
kubectl rollout undo deployment/cosmos-api -n production

# Verify rollback
kubectl get pods -n production
```

---

## Part 8: Monitoring Deployments

### **Check deployment status**
```powershell
kubectl get deployments -n production
kubectl describe deployment cosmos-api -n production
```

### **View pod logs**
```powershell
kubectl logs -f deployment/cosmos-api -n production
```

### **Check image versions deployed**
```powershell
kubectl get pods -n production -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}'
```

### **View deployment history**
```powershell
kubectl rollout history deployment/cosmos-api -n production
```

---

## Part 9: Best Practices

1. **Always test on `develop` first** before merging to `main`
2. **Enable branch protection** on `main`:
   - Require status checks to pass
   - Require code reviews
   - Dismiss stale PR approvals
3. **Monitor Slack notifications** for failures
4. **Use semantic versioning** for releases:
   - Tag: `v1.0.0`, `v1.1.0`, etc.
   - Pipeline auto-creates images: `cosmos-api:1.0.0`
5. **Keep kubeconfig secrets secure**:
   - Rotate them quarterly
   - Use separate service accounts per environment
   - Never commit kubeconfig to git

---

## Part 10: Advanced: GitOps (ArgoCD)

For true GitOps, use ArgoCD to auto-deploy on git commits:

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Create ArgoCD Application
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cosmos-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/JefeEyuel-art-prod/Cosmos-NMV
    targetRevision: main
    path: k8s/
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# Access ArgoCD UI
kubectl port-forward -n argocd svc/argocd-server 8443:443
# https://localhost:8443 (admin / $(kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d))
```

---

## Troubleshooting

### Issue: Deployment fails with "image not found"

```powershell
# Verify image exists on Docker Hub
docker pull eyuel21/cosmos-api:latest

# If missing, rebuild manually
docker build -t eyuel21/cosmos-api:latest ./backend
docker push eyuel21/cosmos-api:latest
```

### Issue: kubectl authentication fails

```powershell
# Verify kubeconfig secret is correct
echo ${{ secrets.KUBE_CONFIG_PROD }} | base64 -d | head -20

# Or test locally
mkdir -p ~/.kube
echo "BASE64_STRING_HERE" | base64 -d > ~/.kube/config
kubectl get namespaces
```

### Issue: Trivy scan reports vulnerabilities

```powershell
# View Trivy results in GitHub Security tab
# Settings → Code security and analysis → Trivy scan results

# Fix: Update base images in Dockerfile
# FROM node:20-alpine → FROM node:22-alpine
```

---

## Commands Cheat Sheet

```powershell
# View workflow runs
gh run list --repo JefeEyuel-art-prod/Cosmos-NMV

# View specific run
gh run view <RUN_ID> --repo JefeEyuel-art-prod/Cosmos-NMV

# Re-run failed job
gh run rerun <RUN_ID> --repo JefeEyuel-art-prod/Cosmos-NMV

# Check deployment status
kubectl get deploy -n production
kubectl rollout status deployment/cosmos-api -n production

# Manual image update
kubectl set image deployment/cosmos-api cosmos-api=eyuel21/cosmos-api:v1.0.0 -n production --record
```

---

**Status**: ✅ Production-grade CI/CD ready

**Next**: Commit, push, and trigger your first automated deployment!
