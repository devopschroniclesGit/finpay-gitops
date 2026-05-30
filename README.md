# finpay-gitops

GitOps repository for FinPay. Argo CD watches this repo and applies changes to the cluster.  
**No application code here. No Terraform here.** Only Kubernetes manifests and Helm values.

## How It Works

```
Developer pushes to finpay-api (GitHub)
        │
        ▼
GitHub Actions CI  ──► Build → Scan → Push image to ECR
        │
        ▼
CI updates image tag   ──► kustomize edit set image ... → git push → HERE
        │
        ▼
Argo CD detects change ──► Applies new Deployment to cluster
        │
        ▼
Kubernetes rolls out   ──► Zero-downtime rolling update
```

## Repo Structure

```
finpay-gitops/
├── projects/
│   ├── finpay-project.yaml    # AppProject (RBAC boundaries)
│   ├── finpay-apps.yaml       # Applications: dev, staging, prod
│   └── infra-apps.yaml        # ingress-nginx, cert-manager, ESO, Prometheus
│
├── apps/
│   └── finpay-api/
│       ├── base/              # Shared manifests (Deployment, Service, HPA, PDB…)
│       └── overlays/
│           ├── dev/           # Patches: 1 replica, small resources, no TLS
│           ├── staging/       # Patches: 2 replicas, medium resources, TLS staging cert
│           └── prod/          # Patches: 3+ replicas, large resources, TLS prod cert
│
└── infrastructure/
    └── argocd/
        ├── app-of-apps.yaml        # Root Application — bootstrap entry point
        ├── argocd-cm.yaml          # Argo CD config
        ├── argocd-rbac-cm.yaml     # RBAC roles
        └── cluster-secret-store.yaml  # ESO ClusterSecretStore (AWS Secrets Manager)
```

## Bootstrap (First-Time Setup)

Assumes EKS cluster exists and Argo CD is installed (see finpay-k8s-infrastructure README).

```bash
# 1. Apply the App-of-Apps — this single command bootstraps everything
kubectl apply -n argocd -f infrastructure/argocd/app-of-apps.yaml

# 2. Watch Argo CD sync all applications
kubectl -n argocd get applications -w

# 3. Access the Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080
```

Argo CD will then install in order (controlled by `sync-wave` annotations):
- **Wave -10**: App-of-Apps root
- **Wave -5**: ingress-nginx, cert-manager, External Secrets Operator
- **Wave -3**: Prometheus stack
- **Wave 0**: finpay-api (dev, staging, prod)

## Making a Change

### Deploying a new app version (automated via CI)
The GitHub Actions pipeline in `finpay-api` does this automatically on push to `main`.  
You never need to manually edit image tags.

### Changing Kubernetes config (e.g. increase replicas)
```bash
# Edit the relevant overlay
vim apps/finpay-api/overlays/prod/kustomization.yaml

# Validate your change renders correctly
kustomize build apps/finpay-api/overlays/prod | kubectl diff -f -

# Commit and push — Argo CD syncs automatically within 3 minutes
git add . && git commit -m "feat(prod): increase min replicas to 5" && git push
```

### Checking what Argo CD will apply (dry-run)
```bash
kustomize build apps/finpay-api/overlays/prod
```

## Environments

| Namespace      | Argo CD App         | Auto-Sync | Image Source               |
|----------------|---------------------|-----------|----------------------------|
| finpay-dev     | finpay-api-dev      | ✅ Yes    | ECR `sha-<commit>`         |
| finpay-staging | finpay-api-staging  | ✅ Yes    | ECR `sha-<commit>`         |
| finpay-prod    | finpay-api-prod     | ❌ No (drift-heal only) | ECR `sha-<commit>` |

Production image tags are updated by CI but the sync itself requires a manual trigger in Argo CD UI or:
```bash
argocd app sync finpay-api-prod
```

## Secrets

Secrets are **never stored here**. They live in AWS Secrets Manager under `finpay/<env>/api`.  
The External Secrets Operator reads them and creates a Kubernetes Secret named `finpay-api-secret`.

To rotate a secret:
```bash
aws secretsmanager put-secret-value \
  --secret-id finpay/prod/api \
  --secret-string '{"JWT_SECRET":"new-value", ...}'
# ESO will pick up the change within 1 hour (refreshInterval)
# The Deployment restarts automatically via the stakater/reloader annotation
```

## Adding a New Environment

1. Copy `apps/finpay-api/overlays/staging` → `apps/finpay-api/overlays/uat`
2. Edit namespace, image registry, resource patches
3. Add an Application entry in `projects/finpay-apps.yaml`
4. Add the namespace to the AppProject destinations in `projects/finpay-project.yaml`
5. Commit and push — Argo CD creates it automatically
