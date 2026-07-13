# AI Task Platform — Infrastructure Repository

GitOps repository for the [AI Task Processing Platform](https://github.com/Dastgir-27/ai-task-platform).
Argo CD watches this repository and automatically syncs the Kubernetes manifests to the
cluster. The application repo's CI pipeline updates image tags here after every build.

## Layout

```
base/                     # shared manifests (Kustomize base)
  configmap.yaml          # non-secret app configuration
  mongo.yaml              # MongoDB StatefulSet + headless Service + PVC
  redis.yaml              # Redis (AOF persistence) + Service + PVC
  backend.yaml            # API Deployment + Service (probes, resources)
  worker.yaml             # Worker Deployment + HorizontalPodAutoscaler
  frontend.yaml           # Frontend Deployment + Service
  ingress.yaml            # Ingress: /api -> backend, / -> frontend
overlays/
  production/             # namespace ai-task-platform, full replicas
  staging/                # namespace ai-task-platform-staging, reduced footprint
argocd/
  application-production.yaml
  application-staging.yaml
```

## Cluster setup (local k3d)

Prerequisites: Docker Desktop, `k3d`, `kubectl`.

```bash
# 1. Create a k3s cluster with the built-in Traefik ingress exposed on localhost:8081
k3d cluster create ai-task --port "8081:80@loadbalancer"

# 2. Create the JWT secret (secrets are never committed to this repo)
kubectl create namespace ai-task-platform
kubectl create secret generic app-secrets \
  --namespace ai-task-platform \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)"

# (repeat for staging if deploying the staging overlay)
kubectl create namespace ai-task-platform-staging
kubectl create secret generic app-secrets \
  --namespace ai-task-platform-staging \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)"
```

> **Note:** GHCR images must be public (GitHub → package → settings → visibility), or
> add an `imagePullSecret` to the deployments.

## Argo CD installation

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods, then register the applications
kubectl apply -f argocd/application-production.yaml
kubectl apply -f argocd/application-staging.yaml

# Access the dashboard
kubectl port-forward svc/argocd-server -n argocd 8443:443
# username: admin — password:
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

Open https://localhost:8443 — both applications should show **Synced / Healthy**
(Auto Sync with prune and self-heal is enabled).

The deployed app is reachable at http://localhost:8081 (Traefik ingress).

## How deployments happen (GitOps flow)

1. A commit lands on `main` in the application repository.
2. GitHub Actions lints, builds, and pushes images to GHCR tagged with the commit SHA.
3. The pipeline runs `kustomize edit set image` in `overlays/production/` of this repo
   and pushes the commit.
4. Argo CD detects the drift and automatically syncs the new image tags to the cluster.

Rollback = `git revert` of the image-bump commit; Argo CD syncs back automatically.

## Verify manifests locally

```bash
kubectl kustomize overlays/production
kubectl kustomize overlays/staging
```
