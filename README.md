# rs-recruiting-gitops

GitOps source of truth for the `rs-recruiting` sandbox environment. ArgoCD
(running in the `rs-recruiting-sandbox` AWS account's EKS cluster) syncs from
this repo; nothing here is applied by hand.

## Layout

| Path | Purpose |
|---|---|
| `envs/sandbox-staging/` | Values overlays for the `sandbox-staging` namespace, synced on every merge to `main` in [`rs-recruiting`](https://github.com/lahavrud/rs-recruiting) |
| `envs/sandbox-prod/` | Values overlays for the `sandbox-prod` namespace, synced on every `v*` tag push in `rs-recruiting` |

Each env has `backend-values.yaml` and `frontend-values.yaml` — overlays for
the Helm charts that live in `rs-recruiting` at `helm/backend/` and
`helm/frontend/`. `image.tag` in each file is the field CI rewrites on
deploy; ArgoCD's `Application` resources (defined in the infra repo, see
below) point at the chart + the matching overlay here.

## Accessing ArgoCD

The cluster is ephemeral — it only exists while someone has applied
`tofu/sandbox` in
[`rs-recruiting-infra`](https://github.com/lahavrud/rs-recruiting-infra). When it's up:

```bash
aws eks update-kubeconfig --region us-east-1 --name rs-sandbox-eks
kubectl get gateway sandbox -n kube-system -o jsonpath='{.status.addresses[0].value}'
# -> http://<that hostname>/argocd/
```

Login is `admin` / the value in SSM `/rs-recruiting/sandbox/ARGOCD_ADMIN_PASSWORD`
(sandbox AWS account):

```bash
aws ssm get-parameter --name /rs-recruiting/sandbox/ARGOCD_ADMIN_PASSWORD \
  --with-decryption --region us-east-1 --query Parameter.Value --output text
```

Grafana is reachable the same way, under `/grafana/`, with credentials in
`/rs-recruiting/sandbox/GRAFANA_ADMIN_PASSWORD`.

## Deploy flow

1. Merge to `main` in `rs-recruiting` → CI builds + pushes images → updates
   `image.tag` in `envs/sandbox-staging/*-values.yaml` here → ArgoCD syncs
   `sandbox-staging`.
2. Push a `v*` tag in `rs-recruiting` → CI updates `envs/sandbox-prod/*-values.yaml`
   here → ArgoCD syncs `sandbox-prod`.

Both directions skip silently if the sandbox SSM flag (`/rs-recruiting/sandbox/active`)
is absent — i.e. if nobody has the cluster up.
