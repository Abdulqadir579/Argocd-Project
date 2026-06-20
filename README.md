# ArgoCD GitOps Lab on Minikube

A local Kubernetes learning environment demonstrating a complete GitOps workflow: deploying applications to a Minikube cluster declaratively from a Git repository using ArgoCD.

## Overview

This project sets up ArgoCD on a local Minikube cluster (macOS) and deploys a sample application (nginx) sourced entirely from Git. ArgoCD continuously reconciles the cluster state against the manifests in the repository, so Git is the single source of truth.

```
Git repo (k8s manifests)  ──>  ArgoCD  ──>  Minikube cluster
```

## Prerequisites

- macOS (tested on Apple Silicon)
- Docker Desktop (running)
- `minikube`
- `kubectl`

## Architecture

ArgoCD runs as a set of pods in the `argocd` namespace:

| Component | Workload Type | Role |
|---|---|---|
| `argocd-application-controller` | StatefulSet | Core reconciliation engine; syncs live state against Git |
| `argocd-server` | Deployment | API / Web UI server |
| `argocd-repo-server` | Deployment | Clones Git repos, renders manifests |
| `argocd-applicationset-controller` | Deployment | Handles ApplicationSet resources |
| `argocd-notifications-controller` | Deployment | Notifications |
| `argocd-dex-server` | Deployment | SSO / auth |
| `argocd-redis` | Deployment | Cache (non-HA install) |

The source of truth is Git plus live cluster state — there are no PVCs in the standard install. The application-controller is a StatefulSet for stable identity (sharding), not for storage.

## Setup

### 1. Start Minikube

```bash
minikube start --cpus 4 --memory 8192 --driver=docker
minikube status
kubectl get nodes
```

### 2. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

Verify all CRDs are present (should show `applications`, `appprojects`, and `applicationsets`):

```bash
kubectl get crd | grep argoproj
```

> **Note:** The `applicationsets` CRD is large. If it is missing and a plain `kubectl apply` fails with a `metadata.annotations: Too long` error, install it with server-side apply:
> ```bash
> kubectl apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/applicationset-crd.yaml
> kubectl rollout restart deployment argocd-applicationset-controller -n argocd
> ```

### 3. Confirm pods are healthy

```bash
kubectl get pods -n argocd
```

All seven pods should be `Running` with no climbing restart counts.

## Repository Structure

The application manifests live in a `k8s/` folder in the Git repo. The ArgoCD `Application` manifest is applied separately and is **not** part of the deployed workload.

```
your-repo/
└── k8s/
    ├── deployment.yaml
    └── service.yaml
```

**`k8s/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: nginx:latest
          ports:
            - containerPort: 80
```

**`k8s/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

> Best practice: leave the `namespace` field out of these manifests and let the Application's `destination.namespace` control placement. Keeps the manifests reusable.

## Deploying the Application

The `Application` object tells ArgoCD what to deploy and where. It lives in the `argocd` namespace; the workload is deployed into the `frontend` namespace.

**`myapp-app.yaml`** (applied with `kubectl`, kept outside the repo's `k8s/` folder):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd          # where the Application object lives
spec:
  project: default           # the ArgoCD AppProject (not a namespace)
  source:
    repoURL: 'https://github.com/Abdulqadir579/YOUR_REPO.git'
    targetRevision: HEAD
    path: k8s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: frontend      # where the workload is deployed
  syncPolicy:
    automated:
      prune: true            # delete resources removed from Git
      selfHeal: true         # revert manual drift back to Git state
    syncOptions:
      - CreateNamespace=true # auto-create the destination namespace
```

Apply and watch:

```bash
kubectl apply -f myapp-app.yaml
kubectl get application myapp -n argocd
kubectl get all -n frontend
```

A healthy result:

```
NAME    SYNC STATUS   HEALTH STATUS
myapp   Synced        Healthy
```

## Accessing the App

```bash
kubectl port-forward svc/frontend -n frontend 8081:80
```

Open http://localhost:8081 — the nginx welcome page confirms end-to-end success.

## Accessing the ArgoCD Dashboard

Port-forward the server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 (accept the self-signed cert warning).

Get the initial admin password (in a separate terminal):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Log in with username `admin` and the printed password.

> The `argocd-initial-admin-secret` exists only for the initial login. After changing the password under User Info, that secret can be deleted.

## Testing the GitOps Loop

To see continuous reconciliation in action:

1. Edit `k8s/deployment.yaml` in the repo — change `replicas: 1` to `replicas: 3`.
2. Commit and push.
3. ArgoCD detects the change (polls roughly every 3 minutes, or click **Refresh** in the UI) and scales the deployment with no manual `kubectl apply`:

```bash
kubectl get pods -n frontend -w
```

Because `selfHeal: true` is set, manually deleting or scaling a resource with `kubectl` is automatically reverted to match Git.

## Key Concepts

- **Two namespaces, two jobs:** the `Application` object lives in `argocd`; the workload it deploys lives in `frontend`.
- **`project: default`** refers to the built-in ArgoCD **AppProject** — it is not a namespace.
- **Large CRDs** (ArgoCD, Istio, cert-manager) need `kubectl apply --server-side` to avoid the client-side annotation size limit.
- **Git is the source of truth.** ArgoCD's job is to make the cluster match the repo.

## Common Commands

```bash
# Cluster lifecycle
minikube start --cpus 4 --memory 8192 --driver=docker
minikube stop
minikube delete

# ArgoCD status
kubectl get pods -n argocd
kubectl get application -n argocd

# App resources
kubectl get all -n frontend

# Logs for a misbehaving controller
kubectl logs -f <pod-name> -n argocd
```

## Next Steps

- Swap `nginx:latest` for a custom application image.
- Use a **Helm** chart or **Kustomize** source instead of plain YAML.
- Adopt the **app-of-apps** pattern to manage multiple Applications declaratively.
