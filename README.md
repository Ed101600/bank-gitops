

# Follow this video to be a ArgoCD Boss
https://youtu.be/JLrR9RV9AFA


# Installing latest/stable version of ArgoCD
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Forward Ports
```
k get services -n argocd
kubectl port-forward service/argocd-server -n argocd 8080:443
```

### Get Credentials
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

# Install ArgoCD CLI / Login via CLI
```
brew install argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login 127.0.0.1:8080
```

# Creating an Application using ArgoCD CLI:
```
argocd app create webapp-kustom-prod \
--repo https://github.com/Ed101600/bank-gitops.git \
--path kustom-webapp/overlays/prod --dest-server https://kubernetes.default.svc \
--dest-namespace 232160-prod
```

# Command Cheat sheet
```
argocd app create #Create a new Argo CD application.
argocd app list #List all applications in Argo CD.
argocd app logs <appname> #Get the application’s log output.
argocd app get <appname> #Get information about an Argo CD application.
argocd app diff <appname> #Compare the application’s configuration to its source repository.
argocd app sync <appname> #Synchronize the application with its source repository.
argocd app history <appname> #Get information about an Argo CD application.
argocd app rollback <appname> #Rollback to a previous version
argocd app set <appname> #Set the application’s configuration.
argocd app delete <appname> #Delete an Argo CD application.
```

# Bank GitOps

**Desired state** for the Bank App platform on EKS: Helm values, Argo CD Applications, Istio policies, and observability installs.

This repo is the source of truth for what runs in the cluster. Humans (and CI) change Git; **Argo CD** makes the cluster match. No production `kubectl apply` for the bank.

## Related repositories

| Repository | Purpose |
| ---------- | ------- |
| [bank-app](https://github.com/Ed101600/bank-app) | Builds images; CI commits digests **into this repo** |
| `bank-infra` | Terraform creates EKS, ECR, RDS, IAM, DNS |
| **bank-gitops** (this) | What Argo CD syncs to the cluster |
| `bank-ansible` | Bastion / host config (outside Kubernetes) |

## What lives here

| Path (target) | Contents |
| ------------- | -------- |
| `apps/` | Argo CD `Application` / `ApplicationSet` manifests |
| `charts/` or `helm/` | Helm charts for bank services (or umbrella chart) |
| `environments/` | `values-dev.yaml`, `values-prod.yaml` (image digests, replicas, URLs) |
| `platform/` | Cluster add-ons as code: ingress, External Secrets, Istio, obs |
| `policies/` | Istio `PeerAuthentication`, `AuthorizationPolicy`, optional NetworkPolicies |

**CI contract:** `bank-app` pipelines push to ECR, then open a commit/PR that only updates image digests/tags under `environments/`. Argo CD reconciles.

## Layout (to be filled in)

```text
bank-gitops/
├── README.md
├── apps/
│   ├── root-app.yaml            # app-of-apps (optional)
│   └── bank-app.yaml
├── helm/
│   └── bank/
│       ├── Chart.yaml
│       ├── templates/
│       └── values.yaml
├── environments/
│   ├── dev/
│   │   └── values.yaml
│   └── prod/
│       └── values.yaml
├── platform/
│   ├── ingress/
│   ├── istio/
│   ├── external-secrets/
│   └── observability/           # Prometheus, Grafana, Fluent Bit, ELK refs
└── policies/
    └── istio-authz.yaml
```

## Bootstrap order

1. `bank-infra` applied → kubeconfig works.  
2. Install Argo CD (bootstrap once; then prefer managing Argo via Git too).  
3. Point a root Application at this repo.  
4. Sync **platform** first (ingress, External Secrets, Istio), then **bank** apps.  
5. Confirm UI via ALB / Route53; mesh mTLS between services.

## Environments

| Env | Cluster | Values | Notes |
| --- | ------- | ------ | ----- |
| `dev` | Dev EKS from `bank-infra` | `environments/dev` | Faster iteration, smaller replicas |
| `prod` | Prod EKS (later) | `environments/prod` | Digests only; stricter AuthZ |

Promote by merging gitops changes (same digests that passed in dev), not by rebuilding differently for prod when avoidable.

## Istio

Multi-service bank → mesh is **in scope** for this project:

- STRICT mTLS in the bank namespace  
- Default-deny `AuthorizationPolicy`, then allow known paths (e.g. frontend → userservice / ledger services)  
- North-south: ALB → Istio Ingress Gateway (or documented Ingress pattern)

Platform team owns mesh install + base policies; app values only expose Service ports and labels Istio needs.

## Secrets

- No raw DB passwords or JWT keys in Git.  
- External Secrets pulls from AWS Secrets Manager (created/referenced in `bank-infra`).  
- Helm mounts synced Kubernetes Secrets.

## Liquibase

Schema migrations ship as a Helm Job or Argo **PreSync** hook using images/changeSets from `bank-app`, targeting **RDS** endpoints from infra outputs—not hand-run SQL on prod.

## Observability (platform folder)

Target stack:

- OpenTelemetry Collector → Prometheus → Grafana (+ Alertmanager)  
- Fluent Bit → Elasticsearch → Kibana (ELK)  
- LogicMonitor optional (collectors often via Ansible on hosts)

## Status

- [ ] Argo CD installed + root app  
- [ ] Helm chart for bank services  
- [ ] Image digests wired from CI  
- [ ] External Secrets + JWT/DB  
- [ ] Ingress / Gateway + DNS  
- [ ] Istio mTLS + AuthZ  
- [ ] Liquibase PreSync/Job  
- [ ] Prometheus/Grafana + ELK  

## License

Learning / personal project. Add a `LICENSE` if you want explicit terms for this GitOps code.
