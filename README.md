# rs-recruiting-gitops

GitOps source of truth for the `rs-recruiting` sandbox environment. ArgoCD
(running in the `rs-recruiting-sandbox` AWS account's EKS cluster) syncs from
this repo; nothing here is applied by hand, except the one bootstrap
Application described below.

## Layout

| Path | Purpose |
|---|---|
| `bootstrap/root.yaml` | The "app of apps" — the only manifest applied imperatively, once, by Tofu (`rs-recruiting-infra`'s `argocd.tf`, via `kubectl`). Points ArgoCD at `apps/` in this repo. |
| `apps/` | One ArgoCD `Application` per backend namespace (`sandbox-dev`/`sandbox-staging`/`sandbox-prod`). The `root` Application above watches this directory — add, remove, or edit an `Application` here and ArgoCD picks it up on its own, no Tofu/`kubectl` involved. |
| `envs/sandbox-dev/`, `envs/sandbox-staging/`, `envs/sandbox-prod/` | `backend-values.yaml` overlay for the matching `Application` in `apps/` — Helm values for the chart at `helm/backend/` in [`rs-recruiting`](https://github.com/lahavrud/rs-recruiting). `image.tag` is the field CI rewrites on deploy. |

Each `Application` is multi-source: one source is the chart itself
(`rs-recruiting` at `helm/backend/`), the other is a `ref` source pointing
back at this repo so the chart source's `helm.valueFiles` can reference the
matching `envs/<namespace>/backend-values.yaml` via `$values/...`.

No frontend overlay — the frontend deploys to S3 + CloudFront instead of
in-cluster (see #18, superseded by `rs-recruiting-infra`'s `cdn.tf`), so
there's no frontend `Application` or values file here.

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
   `image.tag` in `envs/sandbox-dev/backend-values.yaml` and
   `envs/sandbox-staging/backend-values.yaml` here → ArgoCD syncs
   `sandbox-dev` and `sandbox-staging`.
2. Push a `v*` tag in `rs-recruiting` → CI updates
   `envs/sandbox-prod/backend-values.yaml` here → ArgoCD syncs `sandbox-prod`.

Both directions skip silently if the sandbox SSM flag (`/rs-recruiting/sandbox/active`)
is absent — i.e. if nobody has the cluster up.

Runtime secrets (`DATABASE_URL`, `JWT_SECRET_KEY`) never appear in this
repo — each `backend-values.yaml` sets `existingSecret: backend-secrets`,
a Secret Tofu creates per namespace from values it already knows (the RDS
password, a generated JWT secret). See `helm/backend`'s README in
`rs-recruiting` for how the chart consumes it.
