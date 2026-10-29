# lit1537-argocd

GitOps manifests for the `lit-vm` k3s cluster, managed by ArgoCD
(http://argocd.lit1537.infra). Deploys [rCTF](https://github.com/otter-sec/rctf)
with its Kubernetes instancer.

## Layout

| Path | ArgoCD app | Contents |
| --- | --- | --- |
| `bootstrap/` | `root` (apply once manually) | app-of-apps pointing at `apps/` |
| `apps/` | — | ArgoCD Application manifests for the apps below |
| `traefik-config/` | `traefic-config` | HelmChartConfig relocating the k3s traefik release into ns `traefik` + `tcp` entrypoint (1337), plain-HTTP `websecure`, error-pages deployment, `global-errors` Middleware, catch-all IngressRoute |
| `rctf/` | `rctf` | rCTF bundle (Deployment + config), PostgreSQL 18, Redis 8.2, ServiceAccount, IngressRoute for `rctf.lit1537.infra` |
| `rctf-instancer/` | `rctf-instancer` | vendored rctf k8s-operator (`dist/install.yaml` with `INSTANCER_HOST=inst.lit.x3lfy.space`) + ClusterRole/Binding for the rctf ServiceAccount |

## Hostnames / routing

TLS is terminated on the proxy in front of the VM, never on traefik.

- **rCTF UI**: `https://board.lit.x3lfy.space` — proxy terminates TLS and
  forwards plain HTTP (Host preserved) to the node (`100.109.242.58`) on
  port 80 or 443 (both are plain-HTTP entrypoints: `web` / `websecure`).
- **Instances**: `<prefix>-<uid>.inst.lit.x3lfy.space` — instance hostnames
  are one level below `inst.lit.x3lfy.space`.
- **http expose kind (port 80)**: proxy forwards plain HTTP to node:80.
- **https expose kind (port 443)**: proxy terminates TLS and forwards plain
  HTTP (Host preserved) to node:443 — the `websecure` entrypoint is plain
  HTTP on purpose. Direct `http://` hits on node:80 get a 302 to `https://`
  from the operator's redirect route, so the proxy can forward port 80 as-is.
- **tcp-ssl expose kind (port 1337)**: raw TCP passthrough (SNI preserved,
  no TLS termination anywhere in the path — traefik routes by SNI and passes
  the TLS stream through to the challenge pod).

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

## Notes

- After changing `rctf/rctf-config`, restart the app to pick it up:
  `kubectl -n rctf rollout restart deploy/rctf` (config is copied into the
  pod at startup and is not watched).
- Websecure TLS is disabled via the chart values path
  `ports.websecure.http.tls.enabled: false` (note the `http` level).

## Vendored code / custom images

- `rctf` and `rctf-instancer` use images built from the fork
  https://github.com/x3lfyn/rctf (branch `main` = upstream main + fixes),
  published at `ghcr.io/x3lfyn/rctf:main` and `ghcr.io/x3lfyn/rctf/k8s-operator:main`:
  - upstream PR [#235](https://github.com/otter-sec/rctf/pull/235) — API-side
    validation/hashing of the challenge integration id (`inst-<id>-<team>` must
    fit the 63-char namespace name limit)
  - local patch `fix(k8s-operator): bound instance namespace name to 63 chars` —
    operator-side truncate+hash defense so any IDs produce a valid namespace
- `rctf-instancer/operator.yaml` is based on `apps/k8s-operator/dist/install.yaml`
  from the fork with `INSTANCER_HOST` replaced by `inst.lit.x3lfy.space`. Re-vendor
  it when updating the operator.
- `traefik-config/error-pages.yaml` and `traefik-config/traefik-extras.yaml`
  mirror `rctf/deploy/terraform/instancer/modules/k8s/` (traefik.tf,
  error-pages/).
