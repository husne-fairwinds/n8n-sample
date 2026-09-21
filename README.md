# n8n on Kubernetes (Helm values sample)

Sanitized Helm values for running **n8n** in queue mode on Kubernetes: **main** (UI/API), **workers** (executions), and **webhook processors** (production webhooks).

Built against the [official n8n Helm chart](https://github.com/n8n-io/n8n-hosting/tree/main/charts/n8n) (`oci://ghcr.io/n8n-io/n8n-helm-chart/n8n`, pinned below to **1.0.0**). Value keys can change between chart versions — check the chart README before upgrading.

> **Illustrative only.** Hostnames and secret names are placeholders. Do not commit real passwords, encryption keys, or internal DNS.

## Values layout

Layered `-f` files so you can reuse pieces (deps vs sizing vs ingress):

| File | Purpose |
|------|---------|
| [`helm/values-deps.yaml`](helm/values-deps.yaml) | External Postgres + Redis + core secret refs |
| [`helm/values-queue-scale.yaml`](helm/values-queue-scale.yaml) | Queue mode, replicas, HPA |
| [`helm/values-resources.yaml`](helm/values-resources.yaml) | CPU/memory requests and limits |
| [`helm/values-ingress-webhooks.yaml`](helm/values-ingress-webhooks.yaml) | Ingress + `WEBHOOK_URL` |
| [`helm/values-task-runners.yaml`](helm/values-task-runners.yaml) | Optional code-runner sidecars |

## Prerequisites

- Kubernetes 1.25+ and Helm 3.12+
- Reachable **PostgreSQL** and **Redis** (queue mode does not bundle them)
- An ingress controller (example uses `nginx` + cert-manager annotations)
- Kubernetes Secrets for DB password, n8n core env, and (optional) task-runner auth

## Install

```bash
kubectl create namespace n8n

# Required secrets (names match the values files)
kubectl -n n8n create secret generic n8n-db-secret \
  --from-literal=password='CHANGE_ME'

kubectl -n n8n create secret generic n8n-core-secrets \
  --from-literal=N8N_ENCRYPTION_KEY="$(openssl rand -base64 32)" \
  --from-literal=N8N_HOST='n8n.example.com' \
  --from-literal=N8N_PORT='5678' \
  --from-literal=N8N_PROTOCOL='https'

# Optional — only if using values-task-runners.yaml
kubectl -n n8n create secret generic n8n-runner-token \
  --from-literal=auth-token="$(openssl rand -base64 32)"

helm upgrade --install n8n oci://ghcr.io/n8n-io/n8n-helm-chart/n8n \
  --version 1.0.0 \
  -n n8n \
  -f helm/values-deps.yaml \
  -f helm/values-queue-scale.yaml \
  -f helm/values-resources.yaml \
  -f helm/values-ingress-webhooks.yaml
  # add: -f helm/values-task-runners.yaml
```

Edit hosts, Redis/Postgres endpoints, and ingress class before applying.

## Security notes

- Reference Secrets by name; never put credentials in values committed to git  
- Prefer External Secrets / sealed secrets / a vault injector in real environments  
- Keep `N8N_ENCRYPTION_KEY` stable — rotating it without a migration plan can invalidate stored credentials  
