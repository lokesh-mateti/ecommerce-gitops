# 🚀 E-Commerce GitOps Repository

This repository is the **GitOps source of truth** for deploying the [ecommerce-app](https://github.com/lokesh-mateti/ecommerce-app) microservices to AWS EKS using ArgoCD and Helm.

No manual `kubectl apply` is ever run in production — all changes to the cluster flow through this repository.

---

## 🔄 How It Works

```
Jenkins (in ecommerce-app repo)
    │
    │  commits new image tag to values-{env}.yaml
    ▼
ecommerce-gitops (this repo)
    │
    │  ArgoCD watches for changes
    ▼
AWS EKS Cluster
    └── auto-synced within ~3 minutes of every git push
```

---

## 📁 Repository Structure

```
ecommerce-gitops/
├── apps/
│   ├── api-gateway/
│   │   ├── Chart.yaml
│   │   ├── values.yaml           # Base defaults
│   │   ├── values-dev.yaml       # Dev overrides
│   │   ├── values-staging.yaml   # Staging overrides
│   │   ├── values-prod.yaml      # Prod overrides
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       ├── service.yaml
│   │       ├── hpa.yaml
│   │       └── ingress.yaml
│   ├── order-service/            # Same structure
│   └── product-service/          # Same structure
└── argocd/
    ├── app-of-apps.yaml          # Root Application — bootstraps everything
    └── applications/
        ├── api-gateway-app.yaml
        ├── order-service-app.yaml
        └── product-service-app.yaml
```

---

## 🛠️ Helm Charts

Each service has its own Helm chart under `apps/<service-name>/`. The chart is environment-aware — Helm merges `values.yaml` with the environment-specific override file at deploy time.

| Values File | Used By | Purpose |
|---|---|---|
| `values.yaml` | All envs | Base defaults (resource limits, probes, HPA config) |
| `values-dev.yaml` | Dev | Low replicas, no autoscaling, dev ingress host |
| `values-staging.yaml` | Staging | Mid-tier replicas, autoscaling enabled |
| `values-prod.yaml` | Prod | High replicas, aggressive autoscaling, prod ingress host |

### Render a chart locally
```bash
helm template apps/product-service \
  -f apps/product-service/values.yaml \
  -f apps/product-service/values-dev.yaml
```

---

## 🔧 ArgoCD Setup

### Install ArgoCD in EKS
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Bootstrap with app-of-apps (run once)
```bash
kubectl apply -f argocd/app-of-apps.yaml
```

After this single command, ArgoCD manages all three service Applications automatically — any change pushed to this repo is detected and synced to the cluster within minutes.

### Access ArgoCD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
# Default username: admin
# Get password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

---

## 🔐 Secret Management

Secrets (JWT key, DB passwords) are **never stored in this repo**. They are created as Kubernetes Secrets directly in the cluster:

```bash
# Create api-gateway JWT secret before first deploy
kubectl create secret generic api-gateway-secret \
  --from-literal=JWT_SECRET_KEY=<your-strong-secret> \
  -n ecommerce-dev
```

For production, use **External Secrets Operator** or **Sealed Secrets** to manage secrets via GitOps without exposing plaintext values.

---

## 📌 Updating an Image Tag (automated by Jenkins)

Jenkins automatically updates the image tag after a successful pipeline run:

```bash
# This is what Jenkins does — you don't run this manually
sed -i "s|tag:.*|tag: <new-image-tag>|g" apps/product-service/values-dev.yaml
git commit -m "ci: update product-service image to <tag> [dev]"
git push origin main
# ArgoCD picks up the change and deploys automatically
```

---

## 👤 Author

**Lokesh Mateti**
- GitHub: [@lokesh-mateti](https://github.com/lokesh-mateti)
- LinkedIn: [linkedin.com/in/lokesh-mateti](https://linkedin.com/in/lokesh-mateti)
