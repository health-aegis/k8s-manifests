# Aegis Health — Kubernetes Manifests

This repository is the GitOps source of truth for the Aegis Health platform. It contains Helm charts for every service and ArgoCD Application manifests that drive deployments to AKS. The CI/CD pipeline in [health-aegis/health-aegis](https://github.com/health-aegis/health-aegis) writes image tag updates here after each successful build; ArgoCD picks them up and syncs to the cluster automatically.

Infrastructure provisioned by [health-aegis/terraform-aegis](https://github.com/health-aegis/terraform-aegis) must be in place before ArgoCD can deploy anything here.

---

## How It Works

ArgoCD uses the App-of-Apps pattern. A single root Application (`argocd/app-of-apps-dev.yaml` or the prod equivalent) points ArgoCD at `argocd/dev/` or `argocd/prod/`. Each file in those directories is an ArgoCD Application that points at one service's Helm chart in `charts/<service>/` and pulls values from `environments/<env>/values.yaml`.

When the build pipeline pushes a new image to ACR, it also opens a commit on `main` of this repo that bumps the `image.tag` for the affected service in `environments/dev/values.yaml` (or `environments/prod/values.yaml` for releases). ArgoCD detects the diff, applies the Helm chart with the new tag, and the rollout happens without any manual intervention.

```
health-aegis repo push
      |
deploy.yml patches environments/<env>/values.yaml
      |
ArgoCD detects commit on k8s-manifests main
      |
ArgoCD syncs chart with updated image tag
      |
AKS rolling update
```

---

## Repo Structure

```
.
├── argocd/
│   ├── app-of-apps-dev.yaml      # Root ArgoCD Application for dev; points at argocd/dev/
│   ├── app-of-apps-prod.yaml     # Root ArgoCD Application for prod; points at argocd/prod/
│   ├── dev/                      # One ArgoCD Application manifest per service (dev)
│   │   ├── api-gateway-app.yaml
│   │   ├── health-records-service-app.yaml
│   │   ├── medication-service-app.yaml
│   │   ├── ai-service-app.yaml
│   │   ├── imaging-service-app.yaml
│   │   ├── diagnostic-agent-service-app.yaml
│   │   ├── coordinator-agent-app.yaml
│   │   ├── image-analysis-agent-app.yaml
│   │   ├── patient-history-agent-app.yaml
│   │   ├── notification-worker-app.yaml
│   │   ├── client-app.yaml
│   │   └── postgres-app.yaml
│   └── prod/                     # Mirror of dev/ for production namespace
├── charts/
│   ├── api-gateway/              # Helm chart per service
│   ├── health-records-service/
│   ├── medication-service/
│   ├── ai-service/
│   ├── imaging-service/
│   ├── diagnostic-agent-service/
│   ├── coordinator-agent/
│   ├── image-analysis-agent/
│   ├── patient-history-agent/
│   ├── notification-worker/
│   ├── client/
│   └── postgres/
├── environments/
│   ├── dev/
│   │   ├── Chart.yaml
│   │   ├── values.yaml           # Image tags, resource limits, Key Vault config (dev)
│   │   └── templates/
│   │       ├── configmap.yaml    # Shared ConfigMap with inter-service URLs
│   │       ├── serviceaccount.yaml
│   │       ├── secretproviderclass.yaml  # CSI driver SecretProviderClass
│   │       └── ingress.yaml
│   └── prod/                     # Same structure as dev/
└── aegis-health/                 # Umbrella Helm chart (wraps all sub-charts)
```

---

## Prerequisites

The following must be in place on the AKS cluster before applying anything here:

- **ArgoCD** installed in the `argocd` namespace
- **Secrets Store CSI Driver** and the Azure Key Vault provider add-on enabled on AKS
- **Azure Workload Identity** configured on the cluster; the service account `aegis-workload-identity` in `aegis-dev` / `aegis-prod` must be federated to the managed identity provisioned by Terraform
- **AGIC (Application Gateway Ingress Controller)** add-on enabled on AKS
- **Argo Rollouts** installed (used by some services for canary rollout support)
- ACR attached to the AKS cluster (Terraform handles the `AcrPull` role assignment for the kubelet identity, so no image pull secrets are needed)

---

## Setup

### 1. Bootstrap ArgoCD

Install ArgoCD into the cluster if it is not already there:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Get the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### 2. Connect the repo

Either add the repository through the ArgoCD UI (Settings > Repositories) or via CLI:

```bash
argocd repo add https://github.com/health-aegis/k8s-manifests.git \
  --username <github-user> \
  --password <github-pat>
```

### 3. Apply the App-of-Apps

```bash
# Dev
kubectl apply -f argocd/app-of-apps-dev.yaml

# Production
kubectl apply -f argocd/app-of-apps-prod.yaml
```

ArgoCD will discover all Application manifests under `argocd/dev/` (or `argocd/prod/`) and begin syncing each chart.

### 4. Verify

```bash
kubectl get pods -n aegis-dev
kubectl get ingress -n aegis-dev
argocd app list
```

---

## Key Vault Secret Mapping

The `SecretProviderClass` in `environments/dev/templates/secretproviderclass.yaml` pulls these secrets from Azure Key Vault and makes them available as environment variables in every pod via the `aegis-kv-secrets` Kubernetes Secret.

| Key Vault secret name | Environment variable | Consumed by |
|---|---|---|
| `kv-mongodb-uri` | `MONGODB_URI` | api-gateway, health-records, medication, ai-service |
| `kv-jwt-secret` | `JWT_SECRET` | api-gateway |
| `kv-postgres-url` | `DATABASE_URL` | imaging-service |
| `kv-postgres-password` | `POSTGRES_PASSWORD` | postgres StatefulSet |
| `kv-azure-storage-conn` | `AZURE_STORAGE_CONNECTION_STRING` | health-records, imaging-service |
| `kv-gemini-api-key` | `GEMINI_API_KEY` | ai-service, diagnostic-agent-service |
| `kv-azure-ai-endpoint` | `AZURE_AI_ENDPOINT` | health-records-service |
| `kv-azure-ai-key` | `AZURE_AI_KEY` | health-records-service |
| `kv-appinsights-conn` | `APPLICATIONINSIGHTS_CONNECTION_STRING` | all services |
| `kv-servicebus-conn` | `AZURE_SERVICE_BUS_CONNECTION_STRING` | notification-worker |
| `kv-comm-conn-string` | `AZURE_COMM_CONNECTION_STRING` | notification-worker |

The Workload Identity `clientId` and Key Vault name are set in `environments/dev/values.yaml` and get patched automatically by the Terraform pipeline after each `terraform apply` via the `sync-gitops` job in `terraform-core.yml`.

---

## Helm Chart Structure

Each chart under `charts/<service>/` follows the same layout:

```
charts/<service>/
├── Chart.yaml
├── values.yaml       # Defaults (prod-oriented)
├── values-dev.yaml   # Dev overrides (lower resource limits, HPA disabled)
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── service.yaml
    └── networkpolicy.yaml  # Present on agent services
```

The ArgoCD Applications merge both `values.yaml` and `values-dev.yaml` (dev) or only `values.yaml` (prod, since the release pipeline writes to `environments/prod/values.yaml`).

Key values that control a service deployment:

| Value path | Description |
|---|---|
| `image.acrLoginServer` | ACR hostname, e.g. `aegistestacr.azurecr.io` |
| `image.tag` | Image tag; updated by CI/CD pipeline |
| `replicaCount` | Number of pod replicas |
| `resources.requests` / `resources.limits` | CPU and memory per pod |
| `hpa.enabled` | Toggle Horizontal Pod Autoscaler |
| `probes.readiness.path` / `probes.liveness.path` | Health check endpoints |
| `secrets.providerClassName` | Name of the SecretProviderClass to mount |
| `configMap.name` | Name of the shared ConfigMap |

---

## Updating Image Tags Manually

The CI/CD pipeline handles this automatically, but if you need to manually pin a tag:

```bash
# Example: update api-gateway in dev
yq -i '."api-gateway".image.tag = "sha-abc1234"' environments/dev/values.yaml
git add environments/dev/values.yaml
git commit -m "chore: pin api-gateway to sha-abc1234"
git push origin main
```

ArgoCD polls the repo every 3 minutes by default, or you can force an immediate sync:

```bash
argocd app sync api-gateway-dev
```

---

## Namespaces

| Environment | Namespace |
|---|---|
| Dev | `aegis-dev` |
| Production | `aegis-prod` |
| ArgoCD | `argocd` |
