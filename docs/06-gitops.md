# Milestone 5: GitOps and Automated Deployments

## 🎯 **Goal**
Implement GitOps workflows with ArgoCD to automate deployments, ensuring your Kubernetes cluster stays in sync with your Git repository configuration.

## ⏱️ **Typical Time: 30-60 minutes**

## Why This Matters

Manual deployments don't scale. This milestone teaches you how to build deployment pipelines that are reliable, auditable, and can be trusted with production systems.

ℹ️ **Side Note:** GitOps is a methodology where Git is the single source of truth for declarative infrastructure and applications. The basic principle is: "If you want to deploy something, commit it to Git. If you want to rollback, revert the commit." This ensures all changes are tracked, auditable, and can be automatically applied.

## GitOps Flow Overview

```mermaid
graph TB
    subgraph "👨‍💻 Developer Workflow"
        Dev[👨‍💻 Developer]
        Code[📝 Code Changes]
        Config[⚙️ Config Changes]
        Commit[💾 Git Commit]
        Push[🚀 Git Push]
    end
    
    subgraph "📚 Git Repository"
        GitRepo[📚 Git Repository<br/>Single Source of Truth]
        MainBranch[🌿 Main Branch<br/>Production Ready]
        DevBranch[🌱 Dev Branch<br/>Development Work]
        Tags[🏷️ Git Tags<br/>Version Releases]
    end
    
    subgraph "🔄 ArgoCD Controller"
        ArgoCD[🔄 ArgoCD Controller<br/>Watches Git Changes]
        Sync[🔄 Sync Engine<br/>Applies Changes]
        Health[💚 Health Checker<br/>Monitors Status]
        Drift[⚠️ Drift Detection<br/>Finds Differences]
    end
    
    subgraph "☸️ Kubernetes Cluster"
        K8sCluster[☸️ Kubernetes Cluster<br/>Target Environment]
        AppPods[📦 Application Pods<br/>humor-game namespace]
        ConfigMaps[🗂️ ConfigMaps<br/>App Configuration]
        Secrets[🔐 Secrets<br/>Sensitive Data]
        Services[🔌 Services<br/>Network Access]
    end
    
    subgraph "📊 Monitoring & Feedback"
        Prometheus[📊 Prometheus<br/>Metrics Collection]
        Grafana[📈 Grafana<br/>Dashboards]
        Alerts[🚨 Alerts<br/>Status Notifications]
        Logs[📋 Logs<br/>Deployment History]
    end
    
    %% Developer workflow
    Dev --> Code
    Dev --> Config
    Code --> Commit
    Config --> Commit
    Commit --> Push
    Push --> GitRepo
    
    %% Git repository structure
    GitRepo --> MainBranch
    GitRepo --> DevBranch
    GitRepo --> Tags
    
    %% ArgoCD watching Git
    GitRepo --> ArgoCD
    ArgoCD --> Sync
    ArgoCD --> Health
    ArgoCD --> Drift
    
    %% ArgoCD to Kubernetes
    Sync --> K8sCluster
    K8sCluster --> AppPods
    K8sCluster --> ConfigMaps
    K8sCluster --> Secrets
    K8sCluster --> Services
    
    %% Monitoring feedback
    AppPods --> Prometheus
    Prometheus --> Grafana
    Grafana --> Alerts
    K8sCluster --> Logs
    
    %% Styling
    classDef devLayer fill:#e3f2fd
    classDef gitLayer fill:#f3e5f5
    classDef argocdLayer fill:#e8f5e8
    classDef k8sLayer fill:#fff3e0
    classDef monitoringLayer fill:#e0f2f1
    
    class Dev,Code,Config,Commit,Push devLayer
    class GitRepo,MainBranch,DevBranch,Tags gitLayer
    class ArgoCD,Sync,Health,Drift argocdLayer
    class K8sCluster,AppPods,ConfigMaps,Secrets,Services k8sLayer
    class Prometheus,Grafana,Alerts,Logs monitoringLayer
```

## Do This

### Step 1: Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

**Expected Output:**
```bash
namespace/argocd created
```

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

**Expected Output:**
```bash
namespace/argocd created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-server created
configmap/argocd-cm created
configmap/argocd-rbac-cm created
configmap/argocd-tls-certs-cm created
secret/argocd-secret created
deployment.apps/argocd-application-controller created
deployment.apps/argocd-server created
service/argocd-server created
```

# Wait for ArgoCD to be ready (this takes several minutes)
kubectl wait --for=condition=ready pod -l app.kubernetes.io/component=server -n argocd --timeout=300s

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# Save this password - you'll need it to login

**Expected Output:**
```bash
[PLACEHOLDER]
```

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Use your own domain pattern (replace 'gameapp.local' with your domain)
open http://argocd.gameapp.local:8080/
# Login with username: admin, password: (from above command)

**Expected Output:**
```bash
Forwarding from 127.0.0.1:8080 -> 443
Forwarding from [::1]:8080 -> 443
```
```

### Step 2: Create Safe GitOps Structure

**We use a proven safe approach that won't interfere with your working app:**

```bash
# Create GitOps directory structure
mkdir -p gitops-safe/base
mkdir -p gitops-safe/overlays/dev

# Copy your working YAML files (don't modify them!)
cp k8s/configmap.yaml gitops-safe/base/
cp k8s/postgres.yaml gitops-safe/base/
cp k8s/redis.yaml gitops-safe/base/
cp k8s/backend.yaml gitops-safe/base/
cp k8s/frontend.yaml gitops-safe/base/
```

**Repository Structure:**
```
gitops-safe/
├── base/
│   ├── configmap.yaml      # Your working config
│   ├── postgres.yaml       # Your working postgres
│   ├── redis.yaml          # Your working redis
│   ├── backend.yaml        # Your working backend
│   ├── frontend.yaml       # Your working frontend
│   └── kustomization.yaml  # Resource list
└── overlays/
    └── dev/
        └── kustomization.yaml  # Environment overlay
```

### Step 3: Create ArgoCD Project and Application

**Step 3a: Create ArgoCD Project (Security Boundary)**

```yaml
# gitops-safe/argocd-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: humor-game-safe
  namespace: argocd
spec:
  description: "Safe GitOps for Humor Game - Read Only"
  sourceRepos:
  - https://github.com/YOUR_USERNAME/YOUR_REPO
  destinations:
  - namespace: humor-game
    server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
  - group: ""
    kind: ConfigMap
  - group: ""
    kind: PersistentVolumeClaim
  - group: ""
    kind: Service
  - group: apps
    kind: Deployment
  - group: autoscaling
    kind: HorizontalPodAutoscaler
```

**Step 3b: Create ArgoCD Application (Monitoring Only)**

```yaml
# gitops-safe/argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: humor-game-monitor
  namespace: argocd
spec:
  project: humor-game-safe
  source:
    repoURL: https://github.com/YOUR_USERNAME/YOUR_REPO
    targetRevision: gitops
    path: gitops-safe/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: humor-game
  syncPolicy:
    automated: {}  # Manual sync only - SAFE!
    prune: false   # Don't delete anything
```

### Step 4: Apply GitOps Configuration

```bash
# Apply ArgoCD project
kubectl apply -f gitops-safe/argocd-project.yaml

**Expected Output:**
```bash
appproject.argoproj.io/humor-game-safe created
```

# Apply ArgoCD application
kubectl apply -f gitops-safe/argocd-application.yaml

**Expected Output:**
```bash
application.argoproj.io/humor-game-monitor created
```

# Check status
kubectl get applications -n argocd

**Expected Output:**
```bash
NAME                  PROJECT             SYNC STATUS   HEALTH STATUS   AGE
humor-game-monitor    humor-game-safe     OutOfSync     Healthy        2m
```
```

### Step 5: Verify GitOps Setup

**Check ArgoCD UI:**
1. Open `http://argocd.gameapp.local:8080/` (use your own domain)
2. Look for `humor-game-monitor` application
3. Should show **APP HEALTH: Healthy** ✅
4. **SYNC STATUS: OutOfSync** (this is normal and expected)

**Check Resource Count:**
```bash
kubectl describe application humor-game-monitor -n argocd | grep "Kind:" | wc -l
# Should show 12 resources being tracked
```

**Expected Output:**
```bash
12
```

### Step 6: Test GitOps Workflow

**Make a change and see GitOps in action:**
```bash
# Edit any file in gitops-safe/
# Commit and push
git add gitops-safe/
git commit -m "Test GitOps change"
git push origin gitops

# Watch ArgoCD detect the change
kubectl get application humor-game-monitor -n argocd
# Should show OutOfSync status

**Expected Output:**
```bash
NAME                  PROJECT             SYNC STATUS   HEALTH STATUS   AGE
humor-game-monitor    humor-game-safe     OutOfSync     Healthy        5m
```
```

## You Should See...

**ArgoCD Installation:**
```
namespace/argocd created
serviceaccount/argocd-application-controller created
serviceaccount/argocd-server created
deployment.apps/argocd-application-controller created
deployment.apps/argocd-server created
```

**ArgoCD Status:**
```
NAME                 SYNC STATUS   HEALTH STATUS
humor-game-monitor   OutOfSync     Healthy
```

**Resource Count:**
```
12 resources being tracked
```

**ArgoCD UI:**
- Application shows "Healthy" status
- 12 resources being tracked
- "OutOfSync" status (normal - Git and cluster differ)
- Your working app continues running without interruption

## ✅ Checkpoint

Your GitOps workflow is working when:
- ✅ ArgoCD UI shows "Healthy" status
- ✅ 12 resources being tracked
- ✅ "OutOfSync" status (normal - Git and cluster differ)
- ✅ Your working app continues running without interruption

## If It Fails

### Symptom: ArgoCD shows "Unknown" status
**Cause:** ArgoCD can't access your Git repository
**Command to confirm:** Check ArgoCD UI for application status
**Fix:**
```bash
# Check ArgoCD can access your Git repository
# In ArgoCD UI, go to Settings -> Repositories
# Add your repository with proper credentials

# For public repos, use HTTPS URL
# For private repos, add SSH key or access token
```

### Symptom: Application won't sync
**Cause:** Application configuration issues or ArgoCD controller problems
**Command to confirm:** `kubectl describe application humor-game-monitor -n argocd`
**Fix:**
```bash
# Check application configuration
kubectl describe application humor-game-monitor -n argocd

# Check ArgoCD logs
kubectl logs deployment/argocd-application-controller -n argocd

# Manual sync from UI or CLI
argocd app sync humor-game-monitor
```

### Symptom: Kustomization errors
**Cause:** YAML syntax errors or missing files
**Command to confirm:** Test kustomization locally
**Fix:**
```bash
# Test kustomization locally
cd gitops-safe/overlays/dev
kubectl kustomize .

# Check for YAML syntax errors
kubectl apply --dry-run=client -k .
```

## 💡 **Reset/Rollback Commands**

If you need to start over or fix issues:

```bash
# Remove ArgoCD application
kubectl delete application humor-game-monitor -n argocd

# Remove ArgoCD project
kubectl delete appproject humor-game-safe -n argocd

# Reset ArgoCD to factory defaults (nuclear option)
kubectl delete namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Rollback to previous Git commit
git log --oneline -5  # See recent commits
git reset --hard HEAD~1  # Go back 1 commit
git push --force origin gitops  # Force push (be careful!)

# Check ArgoCD status
kubectl get applications -n argocd
kubectl get appprojects -n argocd
```

## Understanding GitOps Benefits

**Compared to manual deployments:**
- **Reliability:** Declarative configuration eliminates deployment steps errors
- **Auditability:** All changes tracked in Git with author and timestamp
- **Rollback:** Easy to revert to any previous Git commit
- **Consistency:** Same deployment process across all environments

**Production GitOps patterns:**
- **Pull-based deployments:** Cluster pulls changes rather than external push
- **Multi-environment promotion:** Automated progression from dev → staging → prod
- **Policy enforcement:** Validation and security scanning before deployment
- **Secret management:** External secret operators for sensitive data

## What You Learned

You've implemented professional GitOps workflows:
- **Declarative deployments** using ArgoCD for automatic synchronization
- **Multi-environment management** with promotion pipelines
- **Audit trails** for all deployment changes through Git history
- **Self-healing deployments** that automatically correct configuration drift
- **Production-ready patterns** used by platform teams at scale

## Professional Skills Gained

- **GitOps architecture** that separates application code from deployment configuration
- **Automated deployment pipelines** that reduce manual errors and deployment time
- **Environment promotion strategies** for safe progression of changes
- **Infrastructure as code** practices that make deployments repeatable and reliable

---

*GitOps milestone completed successfully. ArgoCD running, applications configured, ready for [07-global.md](07-global.md).*
