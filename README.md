# AgriConnect Helm / GitOps

[![ArgoCD Sync](https://img.shields.io/badge/ArgoCD-Synced-brightgreen?logo=argo&logoColor=white)](https://github.com/AgriConnect-Platform/agriconnect-helm)
[![Helm](https://img.shields.io/badge/Helm-v3-0F1689?logo=helm&logoColor=white)](https://helm.sh)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS%201.31-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![AWS](https://img.shields.io/badge/AWS-ap--south--1-FF9900?logo=amazon-aws&logoColor=white)](https://aws.amazon.com)

---

This repository is the **GitOps source of truth** for deploying the AgriConnect microservices platform to AWS EKS. ArgoCD watches this repository and automatically syncs any change to the cluster. Two environments are managed — a `dev` environment (tracking the `dev` branch) and a `prod` environment (tracking the `prod` branch) — both running in the same EKS cluster (`agriconnect-dev-eks`, `ap-south-1`) in separate namespaces.

---

## GitOps Delivery Flow

```
Developer pushes code to agriconnect-app (dev branch)
          │
          ▼
GitHub Actions (main.yml) — CI pipeline runs for all 5 services
    Build → Trivy scan → Smoke test → Push to ECR (SHA tag)
          │
          ▼
update-helm-values job
    Checks out this repo (dev branch)
    sed-replaces image tag in helm/agriconnect/values.yaml
    git push
          │
          ▼
ArgoCD (running in kube-system on agriconnect-dev-eks)
    Detects values.yaml change (polling or webhook)
    Compares desired state vs cluster state
          │
          ▼
ArgoCD syncs — rolling update to all changed Deployments
    (prune=true, selfHeal=true — no manual intervention needed)
          │
          ▼
Kubernetes (namespace: production / dev)
    Pods rolling-updated, HPA still active
    Health checks pass → ALB registers new pods
          │
          ▼
CloudFront → ALB → Kubernetes Ingress → Services → Pods
```

---

## Repository Structure

```
agriconnect-helm/
├── argocd/
│   ├── application.yaml        # ArgoCD app: dev branch → production namespace
│   └── application-prod.yaml   # ArgoCD app: prod branch → prod namespace
├── helm/
│   └── agriconnect/
│       ├── Chart.yaml          # Chart metadata
│       ├── values.yaml         # Default values + dev image tags (ab40be0)
│       ├── values-dev.yaml     # Dev overrides (replicas=1, smaller resources)
│       ├── values-prod.yaml    # Prod overrides (replicas=2, latest tag)
│       └── templates/
│           ├── namespace.yaml          # Namespace
│           ├── serviceaccount.yaml     # IRSA-annotated ServiceAccount
│           ├── configmap.yaml          # Shared env vars for all services
│           ├── ingress.yaml            # AWS ALB Ingress — 5 path rules
│           ├── networkpolicy.yaml      # 4 NetworkPolicy objects (zero-trust)
│           ├── pdb.yaml                # PodDisruptionBudgets for all 5 services
│           ├── auth/
│           │   ├── deployment.yaml     # auth-service Deployment
│           │   ├── hpa.yaml            # HPA: min=2/1(dev), max=6, CPU 70%, Mem 80%
│           │   └── service.yaml        # ClusterIP :3001
│           ├── marketplace/
│           │   ├── deployment.yaml
│           │   ├── hpa.yaml
│           │   └── service.yaml        # ClusterIP :3002
│           ├── order/
│           │   ├── deployment.yaml
│           │   ├── hpa.yaml
│           │   └── service.yaml        # ClusterIP :3003
│           ├── media/
│           │   ├── deployment.yaml
│           │   ├── hpa.yaml
│           │   └── service.yaml        # ClusterIP :3004
│           └── notification/
│               ├── deployment.yaml
│               ├── hpa.yaml
│               └── service.yaml        # ClusterIP :3005
└── override-values.yaml        # Alternative environment overrides (account 978594443309)
```

---

## Chart.yaml

```yaml
apiVersion: v2
name:        agriconnect
description: AgriConnect microservices — all 5 backend services
type:        application
version:     0.1.0
appVersion:  "1.0.0"
```

No subchart dependencies. Single chart deploys all 5 services.

---

## Services — Port and Image Reference

| Service | Port | ECR Image | Dev Tag | Prod Tag |
|---------|------|-----------|---------|----------|
| auth-service | 3001 | `893431614084.dkr.ecr.ap-south-1.amazonaws.com/agriconnect-auth` | `ab40be0` | `latest` |
| marketplace-service | 3002 | `893431614084.dkr.ecr.ap-south-1.amazonaws.com/agriconnect-marketplace` | `ab40be0` | `latest` |
| order-service | 3003 | `893431614084.dkr.ecr.ap-south-1.amazonaws.com/agriconnect-order` | `ab40be0` | `latest` |
| media-service | 3004 | `893431614084.dkr.ecr.ap-south-1.amazonaws.com/agriconnect-media` | `ab40be0` | `latest` |
| notification-service | 3005 | `893431614084.dkr.ecr.ap-south-1.amazonaws.com/agriconnect-notification` | `ab40be0` | `latest` |

All images use `imagePullPolicy: Always`.

---

## Values Configuration

### Global Values (`values.yaml`)

| Key | Value |
|-----|-------|
| `global.namespace` | `production` |
| `global.region` | `ap-south-1` |
| `global.dbSecretName` | `agriconnect/dev/database` |
| `global.eventsTopicArn` | `arn:aws:sns:ap-south-1:893431614084:AgriConnect-Events` |
| `global.notificationsQueueUrl` | `https://sqs.ap-south-1.amazonaws.com/893431614084/AgriConnect-Notifications-Queue` |
| `global.irsaRoleArn` | `arn:aws:iam::893431614084:role/agriconnect-dev-eks-services-role` |
| `global.ecrRegistry` | `893431614084.dkr.ecr.ap-south-1.amazonaws.com` |
| `ingress.enabled` | `true` |
| `ingress.subnets` | `subnet-0040277cc421e9e2c,subnet-0fa1eda6281d6e7b7` |
| `networkPolicy.enabled` | `true` |
| `resources.requests.cpu` | `100m` |
| `resources.requests.memory` | `256Mi` |
| `resources.limits.cpu` | `500m` |
| `resources.limits.memory` | `512Mi` |

### Dev Overrides (`values-dev.yaml`)

| Key | Value |
|-----|-------|
| `global.namespace` | `dev` |
| `resources.requests.cpu` | `50m` |
| `resources.requests.memory` | `128Mi` |
| `resources.limits.cpu` | `250m` |
| `resources.limits.memory` | `256Mi` |
| All services `replicas` | `1` |

### Prod Overrides (`values-prod.yaml`)

| Key | Value |
|-----|-------|
| `global.namespace` | `prod` |
| `global.region` | `ap-south-1` |
| All services `tag` | `latest` |
| All services `replicas` | `2` |
| `resources.requests.cpu` | `100m` (same as default) |
| `resources.limits.cpu` | `500m` (same as default) |

> **Known issue:** `values-prod.yaml` still sets `global.dbSecretName = "agriconnect/dev/database"` and `global.irsaRoleArn` pointing to the dev IAM role. These should be updated to production-specific values before going live.

---

## Replicas and HPA Scaling Matrix

| Service | Replicas (default/prod) | Replicas (dev) | HPA Min | HPA Max | CPU Target | Memory Target |
|---------|------------------------|----------------|---------|---------|------------|---------------|
| auth | 2 | 1 | same as replicas | 6 | 70% | 80% |
| marketplace | 2 | 1 | same as replicas | 6 | 70% | 80% |
| order | 2 | 1 | same as replicas | 6 | 70% | 80% |
| media | 2 | 1 | same as replicas | 6 | 70% | 80% |
| notification | 2 | 1 | same as replicas | 6 | 70% | 80% |

HPA API version: `autoscaling/v2`. `minReplicas` is tied to `.Values.services.<name>.replicas`, so HPA minimum tracks the configured replica count per environment.

---

## Templates — Kubernetes Resources

### `namespace.yaml`

Creates the target namespace:
```yaml
kind: Namespace
metadata:
  name: {{ .Values.global.namespace }}
```
→ `production` (default/prod) or `dev`

### `serviceaccount.yaml`

Single shared `ServiceAccount` for all 5 services with IRSA annotation:

```yaml
kind: ServiceAccount
metadata:
  name: agriconnect-services
  namespace: {{ .Values.global.namespace }}
  annotations:
    eks.amazonaws.com/role-arn: {{ .Values.global.irsaRoleArn }}
```

IAM role `agriconnect-dev-eks-services-role` grants: Secrets Manager (get `agriconnect/*` secrets), SNS Publish, SQS Send/Receive/Delete, S3 on `agriconnect-*` buckets.

### `configmap.yaml`

`ConfigMap` named `agriconnect-config` mounted into all pods via `envFrom`:

| Key | Value |
|-----|-------|
| `NODE_ENV` | `"production"` (hardcoded string — same in all environments) |
| `AWS_REGION` | `ap-south-1` |
| `DB_SECRET_NAME` | `agriconnect/dev/database` |
| `EVENTS_TOPIC_ARN` | `arn:aws:sns:ap-south-1:893431614084:AgriConnect-Events` |
| `NOTIFICATIONS_QUEUE_URL` | `https://sqs.ap-south-1.amazonaws.com/893431614084/AgriConnect-Notifications-Queue` |

DB credentials are NOT in the ConfigMap — they are fetched at runtime from AWS Secrets Manager using `DB_SECRET_NAME` via the IRSA role.

### `ingress.yaml`

AWS ALB Ingress (`ingressClassName: alb`):

| Setting | Value |
|---------|-------|
| Scheme | `internet-facing` |
| Target type | `ip` |
| Listen ports | HTTP:80 only (no TLS configured) |
| Health check path | `/healthz` |
| Health check interval | 30 seconds |
| Healthy threshold | 2 consecutive checks |
| Unhealthy threshold | 3 consecutive checks |
| Subnets | `subnet-0040277cc421e9e2c,subnet-0fa1eda6281d6e7b7` |

**Path routing rules:**

| Path | PathType | Backend Service | Port |
|------|----------|-----------------|------|
| `/api/auth` | Prefix | auth-service | 3001 |
| `/api/marketplace` | Prefix | marketplace-service | 3002 |
| `/api/orders` | Prefix | order-service | 3003 |
| `/api/media` | Prefix | media-service | 3004 |
| `/api/notifications` | Prefix | notification-service | 3005 |

### `networkpolicy.yaml`

Zero-trust network policy — 4 NetworkPolicy objects in the target namespace:

| Policy Name | Selector | Direction | Rule |
|-------------|----------|-----------|------|
| `default-deny-ingress` | all pods `{}` | Ingress | **Deny all** (default deny) |
| `allow-same-namespace` | all pods `{}` | Ingress | Allow from pods in same namespace (inter-service communication) |
| `allow-from-kube-system` | all pods `{}` | Ingress | Allow from `kubernetes.io/metadata.name=kube-system` (ALB controller, CoreDNS, Metrics Server) |
| `allow-all-egress` | all pods `{}` | Egress | Allow all egress (required for AWS SDK: Secrets Manager, S3, SNS, SQS, Bedrock, RDS) |

### `pdb.yaml`

PodDisruptionBudgets for all 5 services (`minAvailable: 1`):

| PDB Name | Pod Selector |
|----------|-------------|
| `auth-service-pdb` | `app: auth-service` |
| `marketplace-service-pdb` | `app: marketplace-service` |
| `order-service-pdb` | `app: order-service` |
| `media-service-pdb` | `app: media-service` |
| `notification-service-pdb` | `app: notification-service` |

### Per-Service Deployments

Each service Deployment shares this configuration pattern:

```yaml
spec:
  replicas: {{ .Values.services.<name>.replicas }}
  template:
    spec:
      serviceAccountName: agriconnect-services
      containers:
        - name: <service>
          image: 893431614084.dkr.ecr.ap-south-1.amazonaws.com/agriconnect-<name>:<tag>
          imagePullPolicy: Always
          ports:
            - containerPort: <port>
          env:
            - name: PORT
              value: "<port>"
          envFrom:
            - configMapRef:
                name: agriconnect-config
          resources:
            requests:
              cpu: 100m        # 50m in dev
              memory: 256Mi    # 128Mi in dev
            limits:
              cpu: 500m        # 250m in dev
              memory: 512Mi    # 256Mi in dev
          livenessProbe:
            httpGet:
              path: /healthz
              port: <port>
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: <port>
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          terminationGracePeriodSeconds: 30
```

> **Note:** No `securityContext` is configured — no `runAsNonRoot`, `readOnlyRootFilesystem`, `runAsUser`, or `capabilities` drop. Containers run with default (likely root) context.

---

## ArgoCD Applications

### Dev Application (`argocd/application.yaml`)

| Field | Value |
|-------|-------|
| `metadata.name` | `agriconnect` |
| `metadata.namespace` | `argocd` |
| `spec.project` | `default` |
| `source.repoURL` | `https://github.com/agriconnect-platform/agriconnect-helm.git` |
| `source.targetRevision` | `dev` |
| `source.path` | `helm/agriconnect` |
| `source.helm.valueFiles` | `values.yaml` |
| `destination.server` | `https://kubernetes.default.svc` |
| `destination.namespace` | `production` |
| `syncPolicy.automated.prune` | `true` |
| `syncPolicy.automated.selfHeal` | `true` |
| `syncOptions` | `CreateNamespace=true`, `ServerSideApply=true` |

### Prod Application (`argocd/application-prod.yaml`)

| Field | Value |
|-------|-------|
| `metadata.name` | `agriconnect-prod` |
| `metadata.namespace` | `argocd` |
| `source.targetRevision` | `prod` |
| `source.helm.valueFiles` | `values-prod.yaml` |
| `destination.namespace` | `prod` |
| All sync settings | Same as dev (automated, prune=true, selfHeal=true) |

Both ArgoCD applications deploy to the **same cluster** (`https://kubernetes.default.svc`) in different namespaces. Both use fully automated sync — any commit to the respective branch triggers a sync within ArgoCD's poll interval.

---

## Manual Operations Reference

### Force Sync an ArgoCD Application

```bash
# Sync the dev app immediately
argocd app sync agriconnect --force --prune

# Sync the prod app
argocd app sync agriconnect-prod --force --prune
```

### Check Application Health

```bash
argocd app get agriconnect
argocd app get agriconnect-prod
```

### Rollback to a Previous Revision

```bash
# List history
argocd app history agriconnect

# Rollback to a specific revision
argocd app rollback agriconnect <revision-number>
```

### Port-Forward a Service (Debugging)

```bash
kubectl port-forward svc/auth-service 3001:3001 -n production
kubectl port-forward svc/marketplace-service 3002:3002 -n production
```

### Check HPA Status

```bash
kubectl get hpa -n production
kubectl describe hpa auth-service-hpa -n production
```

### View Pod Logs

```bash
kubectl logs -l app=auth-service -n production --tail=100
kubectl logs -l app=notification-service -n production --tail=100 -f
```

### Check Network Policies

```bash
kubectl get networkpolicy -n production
kubectl describe networkpolicy default-deny-ingress -n production
```

### Manual Image Tag Update (without CI)

```bash
# Update tag in values.yaml
sed -i 's/tag: ab40be0/tag: <new-sha>/g' helm/agriconnect/values.yaml
git add helm/agriconnect/values.yaml
git commit -m "chore: update image tag to <new-sha>"
git push origin dev
# ArgoCD picks up the change automatically
```

---

## Environment Comparison

| Setting | dev | prod |
|---------|-----|------|
| ArgoCD app name | `agriconnect` | `agriconnect-prod` |
| Helm branch | `dev` | `prod` |
| Values file | `values.yaml` | `values-prod.yaml` |
| Kubernetes namespace | `production` | `prod` |
| Replicas per service | 1 | 2 |
| Image tag | 7-char git SHA | `latest` |
| CPU request | 50m | 100m |
| Memory request | 128Mi | 256Mi |
| CPU limit | 250m | 500m |
| Memory limit | 256Mi | 512Mi |
| HPA min | 1 | 2 |
| HPA max | 6 (all services) | 6 (all services) |

---

## Known Issues and Gaps

| Issue | Location | Impact |
|-------|----------|--------|
| `values-prod.yaml` uses `dbSecretName: "agriconnect/dev/database"` | `values-prod.yaml` | Prod environment reads dev DB credentials |
| `values-prod.yaml` IRSA role still points to `agriconnect-dev-eks-services-role` | `values-prod.yaml` | Prod pods assume dev IAM role |
| `override-values.yaml` uses AWS account `978594443309` (vs `893431614084` everywhere else) | `override-values.yaml` | Undocumented third environment — not wired to any ArgoCD application |
| Ingress HTTP only (port 80) — no TLS | `templates/ingress.yaml` | No HTTPS at ALB layer (HTTPS handled by CloudFront → ALB in HTTP) |
| `NODE_ENV = "production"` hardcoded in ConfigMap regardless of environment | `templates/configmap.yaml` | Dev pods report NODE_ENV=production |
| No securityContext on any container | All deployment templates | Containers run as root by default |
| No ServiceMonitor or PrometheusRules | Entire repo | No Prometheus scraping or alert rules defined here |
| `imagePullPolicy: Always` with `latest` tag in prod | `values-prod.yaml` | Mutable tag in production — no immutable image guarantee |
