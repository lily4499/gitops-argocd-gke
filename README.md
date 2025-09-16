Setup to GitOps your GKE cluster with Argo CD, plus an optional auto-rollback path using Argo Rollouts.

---

# 0) Prereqs (one-time)

```bash
# Local tools
# gcloud (Google Cloud SDK), kubectl, and (optional) argocd CLI installed

# Set GCP context
gcloud auth login
gcloud config set project <YOUR_GCP_PROJECT_ID>
gcloud config set compute/zone us-east4     # pick your preferred zone
```

---

# 1) Create a Git repo with Kubernetes manifests

## A. Scaffold repo locally

```bash
mkdir gitops-argocd-gke && cd gitops-argocd-gke
git init
mkdir -p k8s/base
```

## B. App manifests (simple NGINX example)

`k8s/base/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app
```

`k8s/base/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: app
  labels: { app: web }
spec:
  replicas: 2
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.3
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet: { path: "/", port: 80 }
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: "/", port: 80 }
            initialDelaySeconds: 10
            periodSeconds: 10
```

`k8s/base/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: app
spec:
  type: ClusterIP
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
```

(Optional but handy) `k8s/base/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
```

## C. Commit & push

```bash
git add .
git commit -m "Initial k8s manifests (namespace/deployment/service)"
# Create a GitHub repo first (or any git host) and push:
git branch -M main
git remote add origin https://github.com/<you>/gitops-argocd-gke.git
git push -u origin main
```

---

# 2) Create/Connect a GKE cluster & install Argo CD

## A. Create a GKE cluster (Standard)

```bash
# Standard (node-based) cluster
gcloud container clusters create gke-gitops-demo \
  --num-nodes=2 \
  --machine-type=e2-standard-2

# Fetch kubeconfig
gcloud container clusters get-credentials gke-gitops-demo
kubectl get nodes
```

*(If you prefer Autopilot: `gcloud container clusters create-auto gke-gitops-demo --region us-central1`)*

## B. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd rollout status deploy/argocd-server
```

### Access Argo CD UI (simplest: port-forward)

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:80
# Then open http://localhost:8080
```

### Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d && echo
# Username: admin
```

*(You can also expose the UI via LoadBalancer or Ingress later.)*

---

# 3) Connect Argo CD to your Git repo (Application manifest)

Create an Argo CD `Application` that points to your repo and path.

`argocd-app.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/gitops-argocd-gke.git
    targetRevision: main
    path: k8s/base
  destination:
    server: https://kubernetes.default.svc
    namespace: app
  syncPolicy:
    automated:
      prune: true       # delete resources removed from Git
      selfHeal: true    # if drift happens, Argo re-applies Git state
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Apply it:

```bash
kubectl apply -f argocd-app.yaml
# Watch
kubectl -n argocd get applications.argoproj.io web-app -w
```

Once synced, the `app` namespace will contain your Deployment & Service:

```bash
kubectl -n app get deploy,svc,pods
```

---

# 4) Enable Auto-Sync (already done) & add Auto-Rollback options

You have **auto-sync + self-heal** enabled above. For **automatic rollback on failures**, the simplest reliable approach is to use **Argo Rollouts**, which replaces a standard Deployment with a `Rollout`. If the new version fails to become healthy (readiness doesn’t pass within `progressDeadlineSeconds`, or an analysis check fails), the rollout aborts and **reverts to the previously stable ReplicaSet** (i.e., auto-rollback).

### 4A) Install Argo Rollouts controller

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
kubectl -n argo-rollouts rollout status deploy/argo-rollouts
```

### 4B) Replace Deployment with a Rollout (canary steps, auto-rollback on failure)

**Delete** `k8s/base/deployment.yaml` and **add** `k8s/base/rollout.yaml`:

`k8s/base/rollout.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web
  namespace: app
  labels: { app: web }
spec:
  replicas: 2
  revisionHistoryLimit: 5
  strategy:
    canary:
      # Simple canary: send 20% traffic to new RS, then 100%
      steps:
        - setWeight: 20
        - pause: { duration: 30 } # wait for readiness/observability
        - setWeight: 100
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: nginx
          image: nginx:1.25.3
          ports: [{ containerPort: 80 }]
          readinessProbe:
            httpGet: { path: "/", port: 80 }
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet: { path: "/", port: 80 }
            initialDelaySeconds: 10
            periodSeconds: 10
  progressDeadlineSeconds: 120
```

> **Why this rolls back:** If the new version never becomes healthy within `progressDeadlineSeconds`, the Rollout aborts and reverts to the last stable replicaset automatically.

Commit & push this change:

```bash
git rm k8s/base/deployment.yaml
git add k8s/base/rollout.yaml
git commit -m "Switch Deployment -> Argo Rollout (canary) for auto-rollback"
git push
```

Argo CD will detect the change and sync it. Check status:

```bash
kubectl -n app get rollout web
kubectl -n app describe rollout web
```

*(Optional advanced: wire in metrics AnalysisTemplates with Prometheus or Kayenta to abort/rollback based on SLOs.)*

---

# 5) Operating the flow

* **Change app version**: edit the container image tag in `rollout.yaml` (or `deployment.yaml` if not using Rollouts), commit & push.
  Argo CD auto-syncs → Rollouts performs canary → if healthy, proceeds; if unhealthy, **auto-rollback**.
* **See history**:

  ```bash
  kubectl -n app rollout history rollout/web
  ```
* **Manual override** (rare):

  ```bash
  kubectl -n app argo rollouts undo rollout/web   # requires argo-rollouts kubectl plugin
  ```

---

# 6) (Optional) Expose your app publicly

For a quick test:

```bash
kubectl -n app port-forward svc/web 8080:80
# open http://localhost:8080
```

Or create a LoadBalancer Service:

```yaml
# k8s/base/service.yaml  (change type)
spec:
  type: LoadBalancer
```

Apply via Git (commit & push) → Argo CD syncs → get external IP:

```bash
kubectl -n app get svc web -w
```

---

## Deliverables checklist (ready for your YouTube video)

* **GitHub repo**: `gitops-argocd-gke` with `k8s/base/*` and `argocd-app.yaml`
* **Terraform scripts** *(optional)*: to provision the GKE cluster (you can add later)
* **Screenshots**: Argo CD UI (App → Sync → Healthy), Rollouts details (if enabled)
* **Outcome**: “Reduced manual deployment time by \~60–70%, standardized Git-driven releases, and automatic rollback on failures with Argo Rollouts.”

---
---

**Rollout mechanism** in **Argo Rollouts** step by step, so you can see exactly how it works compared to a plain Kubernetes Deployment.

---

# 🚀 What is a Rollout?

* A **Rollout** is a **drop-in replacement** for a standard Kubernetes **Deployment**.
* It adds **advanced deployment strategies**:

  * Canary
  * Blue-Green
  * Progressive delivery with metrics checks
  * Auto-promotion or manual approval
  * Auto-rollback if health checks fail

---

# ⚙️ How Rollout Mechanism Works

### 1. **Deployment (vanilla K8s)**

* A Deployment updates pods in-place by creating a new ReplicaSet and scaling it up while scaling the old one down (rolling update).
* If something fails (bad image, crash loops), you must **manually rollback**.

---

### 2. **Rollout (Argo Rollouts CRD)**

* A Rollout **replaces Deployment** but introduces **fine-grained control** over ReplicaSets and traffic shifting.
* Instead of immediately scaling new pods, it follows the defined **strategy**.

---

## 🟢 Canary Strategy (1 Service)

* **One Service** routes traffic to pods.
* Rollout gradually increases traffic weight to the new ReplicaSet (e.g., 20% → 50% → 100%).
* If pods fail readiness probes or custom analysis (e.g., Prometheus metrics), Rollout **pauses or auto-aborts**.
* On abort → old ReplicaSet stays active (**auto-rollback**).

---

## 🔵 Blue-Green Strategy (2 Services)

* **Active Service** routes all production traffic.
* **Preview Service** points to the new ReplicaSet (green).
* You test green while blue is still serving traffic.
* At cutover, Rollout switches Active Service to green.
* If the green version fails, Rollout automatically reverts back to the old ReplicaSet (**rollback**).

---

## 🔄 Auto-Rollback

* Rollout monitors pod health, readiness probes, and (optionally) **AnalysisTemplates** (Prometheus/Kayenta).
* If health criteria fail:

  * New ReplicaSet is scaled down.
  * Old stable ReplicaSet is scaled back up.
  * ✅ End users are shielded — they keep hitting the stable version.

---

# 📊 Rollout Lifecycle

1. **New version pushed to Git** → Argo CD syncs manifests.
2. **Rollout creates a new ReplicaSet** for the new version.
3. **Traffic shifting (Canary)** or **Service switching (Blue-Green)** happens.
4. **Health checks / Analysis** are run.
5. **Success** → new RS promoted as stable.
6. **Failure** → rollback to old RS.

---

👉 In short:

* **Deployment** = blind rolling update, manual rollback.
* **Rollout** = controlled rollout (Canary/Blue-Green), safe auto-rollback, GitOps-friendly.

---




