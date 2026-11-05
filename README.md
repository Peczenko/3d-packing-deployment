# 3d-packing-deployment

Kubernetes manifests for the 3D packing platform: the frontend, the Spring API, the Go packing worker and a single-process Temporal with its own Postgres. A k3s node syncs this repository with Flux; nothing is deployed from a laptop.

```
browser ─► https://3dpacking.dev ─► Traefik :443 ─┬─ /api, /swagger-ui, /v3/api-docs ─► api ─┐ gRPC
           http://…, https://www.… ─► 301           └─ /* ─► frontend                        ├──► temporal ──► temporal-postgres
                                          packing-worker (Go, RunEngine) ─────────────────────┘
api ──► Azure Postgres     api, worker ──► Azure Blob     External Secrets Operator ──► Azure Key Vault
```

## Layout

- `base/platform` — Traefik's Gateway API provider and External Secrets Operator, applied first because they install CRDs.
- `base/app` — namespace `packing`: `ClusterSecretStore` and `ExternalSecret`s, Temporal, its Postgres and UI, api, frontend, worker, Gateway and HTTPRoutes, NetworkPolicies (default deny).
- `overlays/gcp` — image tags, TLS on 443, `http` → `https` and `www` → apex redirects, Temporal's PVC on `local-path`.
- `clusters/gcp` — Flux (source and kustomize controllers): `platform` applies `base/platform`, `app` depends on it and applies `overlays/gcp`. Both prune.

## Deploying

Change `newTag` in `overlays/gcp/kustomization.yaml` and push to `main`. Flux polls every minute and rolls the Deployments. Backend images come from releases of `Peczenko/3d-packing-backend`, the frontend's from the `demo` branch of `furby73/3d-packing-frontend`.

## Not in this repository

Created on the node once, never pruned by Flux:

- Secret `packing/azure-sp` — `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` of the service principal ESO, the api and the worker use.
- Secret `packing/packing-tls` — certificate and key for `3dpacking.dev` and `www.3dpacking.dev`.
- ConfigMap `packing/frontend-config` — `config.js` with the Firebase web config.

Everything else the apps need is synced from Key Vault by ESO.
