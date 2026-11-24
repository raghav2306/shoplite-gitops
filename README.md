# shoplite-gitops

The **single source of truth** for what runs in the ShopLite EKS cluster.
Argo CD watches this repo and makes the cluster match it.

| Repo | Role |
|---|---|
| [shoplite-infra](https://github.com/raghav2306/shoplite-infra) | Terraform: EKS, ECR, CI role, and Argo CD with the `shoplite-eks` Application that points here |
| [shoplite-microservices](https://github.com/raghav2306/shoplite-microservices) | App code. Its CI pushes images and commits the new tags to this repo |
| **shoplite-gitops** (this one) | Kubernetes manifests, synced by Argo CD |

Don't `kubectl apply` or `kubectl edit` resources Argo CD tracks: self-heal
reverts them. Change the YAML here, commit and push.

## Layout

Kustomize: a cluster-neutral **base** and the **eks** overlay that adds the
AWS-specific parts. Everything runs in the `ecommerce` namespace.

```
├── base/                          never applied on its own
│   ├── kustomization.yaml         lists every resource below
│   ├── namespace.yaml             namespace "ecommerce"
│   ├── mongodb-users/             StatefulSet + headless Service + secret template
│   ├── mongodb-products/          StatefulSet + headless Service + secret template
│   ├── mongodb-orders/            StatefulSet + headless Service + secret template
│   ├── rabbitmq/                  StatefulSet + ClusterIP & headless Services + secret template
│   ├── user-service/              Deployment + Service + ConfigMap + secret template
│   ├── product-service/           Deployment + Service + ConfigMap
│   ├── order-service/             Deployment + Service + ConfigMap
│   ├── api-gateway/               Deployment + Service + ConfigMap
│   ├── storefront/                Deployment + Service
│   └── admin/                     Deployment + Service
└── overlays/
    └── eks/                       what Argo CD deploys
        ├── kustomization.yaml     ECR images + tags (set by CI), ebs-gp3 patch, ALB health check
        ├── storageclass.yaml      encrypted gp3 EBS class "ebs-gp3"
        └── ingress.yaml           internet-facing ALB: /api, /admin, /
```

## What gets deployed

| Component | Kind | Replicas | Port | Health checks | Talks to |
|---|---|---|---|---|---|
| storefront | Deployment | 2 | 80 | HTTP `/healthz` | — (browser calls `/api` through the ALB) |
| admin | Deployment | 2 | 80 | HTTP `/healthz` | — (same) |
| api-gateway | Deployment | 2 | 3000 | HTTP `/health` | user-, product-, order-service (gRPC) |
| user-service | Deployment | 2 | 50051 gRPC | gRPC health (`liveness` / readiness = Mongo up) | mongodb-users |
| product-service | Deployment | 2 | 50052 gRPC | gRPC health | mongodb-products |
| order-service | Deployment | 2 | 50053 gRPC | gRPC health | mongodb-orders, product-service, RabbitMQ (publishes order events; nothing consumes them yet) |
| mongodb-users / -products / -orders | StatefulSet | 1 each | 27017 | `mongosh ping` | — (1 GiB gp3 volume each) |
| rabbitmq | StatefulSet | 1 | 5672, 15672 UI | `rabbitmq-diagnostics` | — (1 GiB gp3 volume) |

Only the Ingress is public: `/api` → api-gateway, `/admin` → admin, `/` →
storefront. Everything else is a ClusterIP Service.

What `overlays/eks` adds to `base`:

| | `base` | `overlays/eks` |
|---|---|---|
| Images | placeholder `shoplite/<name>` | `<account>.dkr.ecr.<region>.amazonaws.com/dev/<name>:<commit-sha>` |
| Storage | cluster default class | `ebs-gp3` for every StatefulSet (+ the StorageClass) |
| Ingress | none | ALB (`ingressClassName: alb`) |
| Gateway health check | — | ALB annotation `/health` on the api-gateway Service |

Put changes that apply everywhere in `base/` and AWS-specific ones in
`overlays/eks/`. Check the result before you push:

```bash
kubectl kustomize overlays/eks
```

## How a release gets here

1. A push to `main` in shoplite-microservices runs CI: build, scan, push all 6
   images to ECR tagged with the commit SHA.
2. CI's `deploy` job runs `kustomize edit set image` in `overlays/eks` and
   commits `deploy: shoplite-microservices@<sha> to eks` here.
3. Argo CD notices the commit (it polls every ~3 min; REFRESH in the UI to skip
   the wait) and does a rolling update.

ECR tags are immutable, so a tag in this repo always means the same image.
The `573802369594…:v1` entries in `overlays/eks` are placeholders until the
first CI run replaces them with your account's registry and a commit SHA.

**Roll back** by reverting the deploy commit. Don't use `kubectl rollout undo`:
Argo CD would put the newer version back.

```bash
git log --oneline -- overlays/eks/kustomization.yaml
git revert <sha> && git push
```

## Secrets

Secrets are **not** in base or the overlay, so their values never reach Git
and Argo CD never manages or prunes them. Copy the templates and fill in real
values (the `secret.yaml` copies are git-ignored):

| Secret | Template | Used by |
|---|---|---|
| `mongodb-users-secret` | `base/mongodb-users/` | mongodb-users, user-service |
| `mongodb-products-secret` | `base/mongodb-products/` | mongodb-products, product-service |
| `mongodb-orders-secret` | `base/mongodb-orders/` | mongodb-orders, order-service |
| `rabbitmq-secret` | `base/rabbitmq/` | rabbitmq, order-service |
| `user-service-secret` | `base/user-service/` | user-service (`JWT_SECRET`) |

```bash
for d in mongodb-users mongodb-products mongodb-orders rabbitmq user-service; do
  cp base/$d/secret.yaml.example base/$d/secret.yaml
done
```

- MongoDB and RabbitMQ passwords: letters and digits only, since they're embedded in connection URLs.
- `JWT_SECRET`: `openssl rand -hex 32`.
- MongoDB and RabbitMQ read their passwords **only on first start**, when the volume is empty, so set them before the first deploy.

`scripts/apply.sh` in shoplite-infra applies them to the cluster. To do it by hand:

```bash
kubectl create namespace ecommerce
for d in mongodb-users mongodb-products mongodb-orders rabbitmq user-service; do
  kubectl apply -f base/$d/secret.yaml
done
```

A missing Secret shows up as `CreateContainerConfigError` on the pods that
need it. They recover by themselves once it exists.

## Adding a service

1. Create `base/<name>/` with a Deployment (image `shoplite/<name>`), a Service
   if anything calls it, and a ConfigMap. Copy order-service or product-service.
2. List the files in `base/kustomization.yaml`.
3. Add `<name>` to the image list in shoplite-microservices CI
   (`.github/workflows/ci-cd.yml`, the `images` matrix and the `deploy` loop),
   and to `app_images` in shoplite-infra's `ecr.tf` so its ECR repo exists.
4. `kubectl kustomize overlays/eks`, commit, push.
