# finpay-gitops

GitOps repository for FinPay microservices on AWS EKS.
ArgoCD watches this repo and applies all changes to the cluster.
**No application code here. No Terraform here.** Only Kubernetes manifests.

## How It Works

```
Developer pushes to finpay-microservices (GitHub)
        │
        ▼
GitHub Actions CI  ──► Build 4 images → Push to ECR
        │
        ▼
CI updates image tags  ──► kustomize edit set image → git push → HERE
        │
        ▼
ArgoCD detects change  ──► Applies Deployments to cluster
        │
        ▼
Kubernetes rolls out   ──► Zero-downtime rolling update (maxUnavailable: 0)
```

## Repo Structure

```
finpay-gitops/
├── apps/
│   ├── finpay-api/
│   │   ├── base/
│   │   │   ├── deployment.yaml          # Legacy finpay-api (to be removed)
│   │   │   ├── service.yaml
│   │   │   ├── externalsecret.yaml      # ESO — pulls from AWS Secrets Manager
│   │   │   ├── hpa.yaml
│   │   │   ├── pdb.yaml
│   │   │   ├── serviceaccount.yaml
│   │   │   ├── networkpolicy.yaml
│   │   │   ├── namespace.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── services/
│   │   │       ├── auth-deployment.yaml         # auth-service :3001
│   │   │       ├── account-deployment.yaml      # account-service :3002
│   │   │       ├── transaction-deployment.yaml  # transaction-service :3003
│   │   │       └── notification-deployment.yaml # notification-service :3004
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
│   │           ├── kustomization.yaml   # Image tags, replicas, resource patches
│   │           └── ingress.yaml         # ALB routing by path to each service
│   └── finpay-web/
│       └── base/
│           ├── deployment.yaml          # nginx serving React build
│           └── kustomization.yaml
│
└── infrastructure/
    ├── argocd/
    │   ├── argocd-ingress.yaml          # ALB ingress for ArgoCD UI
    │   ├── cluster-secret-store.yaml    # ESO ClusterSecretStore
    │   ├── argocd-cm.yaml
    │   └── argocd-rbac-cm.yaml
    └── monitoring/
        └── grafana-ingress.yaml         # ALB ingress for Grafana UI
```

## ArgoCD Applications

| App | Path | Namespace | Status |
|---|---|---|---|
| `finpay-prod` | `apps/finpay-api/overlays/prod` | finpay | Auto-sync, self-heal |
| `finpay-web` | `apps/finpay-web/base` | finpay | Auto-sync, self-heal |

## ALB Ingress Routing

```
ALB (finpay-ingress)
├── /api/v1/auth         → auth-svc:3001
├── /api/v1/accounts     → account-svc:3002
├── /api/v1/transactions → transaction-svc:3003
└── /api/v1/health       → auth-svc:3001
```

## Bootstrap (First-Time Setup)

ArgoCD is installed by Terraform (`finpay-eks-infra`). After `terraform apply`:

```bash
# Applied automatically by post-apply.sh — or run manually:

kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finpay-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/devopschroniclesGit/finpay-gitops
    targetRevision: main
    path: apps/finpay-api/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: finpay
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finpay-web
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/devopschroniclesGit/finpay-gitops
    targetRevision: main
    path: apps/finpay-web/base
  destination:
    server: https://kubernetes.default.svc
    namespace: finpay
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

## Apply Platform Ingresses

```bash
# ArgoCD and Grafana get their own ALBs
kubectl apply -f infrastructure/argocd/argocd-ingress.yaml
kubectl apply -f infrastructure/monitoring/grafana-ingress.yaml

# Get all ALB URLs
kubectl get ingress -A
```

## Making a Change

### Deploy a new image version

```bash
# CI does this automatically on merge to main in finpay-microservices
# Manual update:
cd apps/finpay-api/overlays/prod
kustomize edit set image \
  finpay-auth=150103290775.dkr.ecr.us-east-1.amazonaws.com/finpay-auth:abc1234

git add . && git commit -m "chore: update auth image to abc1234" && git push
# ArgoCD syncs within 3 minutes
```

### Change Kubernetes config

```bash
# Edit the relevant file
vim apps/finpay-api/overlays/prod/kustomization.yaml

# Validate locally
kubectl kustomize apps/finpay-api/overlays/prod

# Commit and push — ArgoCD syncs automatically
git add . && git commit -m "feat: increase replicas to 3" && git push
```

### Force an immediate sync

```bash
kubectl annotate application finpay-prod -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

## Secrets

Secrets are **never stored in this repo**. They live in AWS Secrets Manager at `finpay/production`.

The External Secrets Operator reads them and creates K8s Secret `finpay-api-secret` in the `finpay` namespace.

| Secret key | Description |
|---|---|
| `DATABASE_URL` | PostgreSQL connection string (`!` encoded as `%21`) |
| `REDIS_URL` | ElastiCache endpoint |
| `JWT_SECRET` | Token signing key |
| `RABBITMQ_URL` | AMQP connection string (`!` encoded as `%21`) |

To rotate a secret:

```bash
aws secretsmanager update-secret \
  --secret-id finpay/production \
  --region us-east-1 \
  --secret-string '{"JWT_SECRET":"new-value", ...}'

# Force ESO to refresh immediately
kubectl annotate externalsecret finpay-api-secret -n finpay \
  force-sync=$(date +%s) --overwrite
```

## Checking ArgoCD status

```bash
# CLI
kubectl get applications -n argocd

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# UI — get ALB URL
kubectl get ingress argocd-ingress -n argocd
```

## Known issues

| Issue | Status |
|---|---|
| Old `finpay-api` deployment still in base | Should be removed — still running alongside microservices |
| notification-service 54+ restarts | RabbitMQ reconnect logic needed |
| ALB URLs change after every rebuild | Add Route53 custom domain |

## Related repos

| Repo | Purpose |
|---|---|
| [finpay-microservices](https://github.com/devopschroniclesGit/finpay-microservices) | 4 Node.js microservices — source of Docker images |
| [finpay-eks-infra](https://github.com/devopschroniclesGit/finpay-eks-infra) | Terraform — provisions EKS, RDS, Redis, ECR, ArgoCD |
| [finpay-web](https://github.com/devopschroniclesGit/finpay-web) | React + Vite frontend |
| [finpay-api](https://github.com/devopschroniclesGit/finpay-api) | Original monolith |
| [finpay-infrastructure](https://github.com/devopschroniclesGit/finpay-infrastructure) | Original Elastic Beanstalk setup |
