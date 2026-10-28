# lit1537-argocd

GitOps manifests for the `lit-vm` k3s cluster, managed by ArgoCD
(http://argocd.lit1537.infra). Deploys [rCTF](https://github.com/otter-sec/rctf)
with its Kubernetes instancer.

## Layout

| Path | ArgoCD app | Contents |
| --- | --- | --- |
| `bootstrap/` | `root` (apply once manually) | app-of-apps pointing at `apps/` |
| `apps/` | — | ArgoCD Application manifests for the apps below |
| `traefik-config/` | `traefic-config` | HelmChartConfig relocating the k3s traefik release into ns `traefik` + `tcp` entrypoint (1337), error-pages deployment, `global-errors` Middleware, catch-all IngressRoute, default TLSStore |
| `rctf/` | `rctf` | rCTF bundle (Deployment + config), PostgreSQL 18, Redis 8.2, ServiceAccount, IngressRoute for `rctf.lit1537.infra` |
| `rctf-instancer/` | `rctf-instancer` | vendored rctf k8s-operator (`dist/install.yaml` with `INSTANCER_HOST=inst.lit.x3lfy.space`) + ClusterRole/Binding for the rctf ServiceAccount |

## Hostnames / routing

- **rCTF UI**: `http://rctf.lit1537.infra` — served by the cluster traefik
  (entrypoint `web`). Add `100.109.242.58 rctf.lit1537.infra` to `/etc/hosts`
  (same pattern as `argocd.lit1537.infra`).
- **Instances**: `<prefix>-<uid>.inst.lit.x3lfy.space` — a wildcard proxy
  outside the cluster forwards `*.inst.lit.x3lfy.space` (ports 80/443/1337,
  **TCP passthrough**, preserving Host/SNI) to the node's tailscale IP
  `100.109.242.58`. The proxy must be L4: the operator hardcodes the published
  ports (http → 80, https → 443, tcp-ssl → 1337) and https/tcp routes are
  matched by SNI/Host at traefik.
- HTTPS instances terminate TLS at traefik with the default TLSStore
  (`instancer-wildcard-tls` in ns `traefik`, self-signed — expect a browser
  warning; replace the secret with a real `*.inst.lit.x3lfy.space` cert to fix).

## Traefik notes

The rctf k8s-operator hardcodes that traefik runs in namespace `traefik`
(its per-instance NetworkPolicies only allow ingress from pods labeled
`app.kubernetes.io/name: traefik` in that namespace). Since k3s's bootstrap
traefik lives in `kube-system` and owns ports 80/443, the
`traefik-config/helmchartconfig.yaml` relocates the k3s-managed release into
ns `traefik` (via the chart's `namespaceOverride`) and adds the `tcp`
entrypoint on 1337. This survives k3s restarts because HelmChartConfig is a
separate CR that k3s merges into the bootstrap chart on every reconcile.

Existing `argocd-server` IngressRoute keeps working: the relocated traefik
still watches IngressRoutes in all namespaces.

## Secrets (out-of-band, NOT in this public repo)

`rctf/rctf-secrets` in ns `rctf`:

```sh
kubectl -n rctf create secret generic rctf-secrets \
  --from-literal=database-password="$(openssl rand -hex 24)" \
  --from-literal=redis-password="$(openssl rand -hex 24)" \
  --from-literal=token-key="$(openssl rand -base64 32)"
```

`traefik/instancer-wildcard-tls` in ns `traefik` (self-signed wildcard):

```sh
openssl req -x509 -newkey rsa:2048 -nodes -days 825 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=*.inst.lit.x3lfy.space" \
  -addext "subjectAltName=DNS:*.inst.lit.x3lfy.space,DNS:inst.lit.x3lfy.space"
kubectl -n traefik create secret tls instancer-wildcard-tls --cert=tls.crt --key=tls.key
```

## Vendored code

- `rctf-instancer/operator.yaml` is `apps/k8s-operator/dist/install.yaml` from
  https://github.com/otter-sec/rctf (image `ghcr.io/otter-sec/rctf/k8s-operator:latest`)
  with `INSTANCER_HOST` replaced by `inst.lit.x3lfy.space`. Re-vendor it when
  upgrading the operator.
- `traefik-config/error-pages.yaml` and `traefik-config/traefik-extras.yaml`
  mirror `rctf/deploy/terraform/instancer/modules/k8s/` (traefik.tf,
  error-pages/).
