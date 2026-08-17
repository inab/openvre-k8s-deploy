# Deploy OpenVRE with interactive RStudio (new cluster)

This chart is **`openvre-k8s-deploy` + interactive addons**.  
Your plain `openvre-k8s-deploy` is **not enough** for interactive sessions.

## What the base chart was missing

| Item | Base `openvre-k8s-deploy` | This chart |
|------|---------------------------|------------|
| Scheduler RBAC for Deploy/Ingress/CM | pods list only | full interactive RBAC |
| `OPENVRE_EXTERNAL_BASE` on scheduler | no | yes |
| Disable legacy Apache `/interactive-tool` proxy | always on (breaks K8s path) | configurable off |
| Mongo seed for `rstudio` tool | no | `tools-rstudio.json` |
| RStudio tool UI files (`tools/front/rstudio/`) | no | bundled under `files/tools/rstudio/` → seed to PVC `tools/front/rstudio/` |
| Frontend PHP interactive code | **not in Helm** — must be in **frontend image** | build from `openvre-dev-kubernetes-interactive-rstudio-live` |
| Scheduler `app.py` interactive API | **not in Helm** — must be in **scheduler image** | use `scheduler-3.1` |

## Prerequisites

1. **NGINX Ingress Controller** in the cluster.
2. Images built and pushed:

```bash
# Frontend (includes ProcessK8sInteractive.php, Tooljob, actions-home.js, processJob.inc.php)
cd /path/to/openvre-dev-kubernetes-interactive-rstudio-live/openVRE-core-dev/front_end
docker build -t ymaqsoodbsc/openvre-kubernetes:frontend-interactive-rstudio-1 .
docker push ymaqsoodbsc/openvre-kubernetes:frontend-interactive-rstudio-1

# Scheduler (build ONLY from scheduler/ directory)
cd /path/to/openvre-dev-kubernetes-interactive-rstudio-live/openVRE-core-dev/scheduler
docker build -t ymaqsoodbsc/openvre-kubernetes:scheduler-3.1 .
docker push ymaqsoodbsc/openvre-kubernetes:scheduler-3.1

# RStudio session image
cd /path/to/openvre-dev-kubernetes-interactive-rstudio-live/openVRE-core-dev/rstudio-image
docker build -f Dockerfile-rstudio.txt -t ymaqsoodbsc/openvre-kubernetes:rstudio-init-wrapper-2 .
docker push ymaqsoodbsc/openvre-kubernetes:rstudio-init-wrapper-2
```

3. Edit `files/tools/rstudio/mongo-k8s.json` if you use a different RStudio image tag.

## Helm install

```bash
cd /path/to/openvre-k8s-deploy-interactive
cp my-values-interactive.example.yaml my-values.yaml
# edit my-values.yaml (secrets, domain.host, image tags)

helm upgrade --install openvre . -n openvre --create-namespace -f my-values.yaml
```

## After install (important)

Mongo seeds `rstudio` only on **first empty** Mongo volume.  
If Mongo already exists, import manually:

```bash
kubectl -n openvre exec deploy/dashboard-mongodb -- mongosh ...
# or re-use tools-rstudio.json with mongoimport
```

Copy RStudio tool UI into frontend tools PVC:

```bash
chmod +x scripts/seed-rstudio-tool-files.sh
./scripts/seed-rstudio-tool-files.sh openvre
```

## Verify

```bash
kubectl -n openvre get deploy scheduler dashboard-frontend
kubectl -n openvre logs deploy/scheduler --tail=20
# Launch RStudio from UI → URL like /interactive-tool/<session-id>/
```

### Interactive session URL returns Apache 404

The main app Ingress uses host `domain.host` with path `/` (catch-all). Session Ingresses **must** use the same host (from `scheduler.externalBase`, e.g. `http://openvre.local`). Without it, `/interactive-tool/<id>/` hits Apache and returns 404.

With `interactive.disableLegacyApacheProxy: true` (default), Apache proxies `/interactive-tool/<session-id>/` to Service `openvre-ix-<session-id>:8787` — works for every new session, no scheduler change.

Optional: `interactive.ingressSync.enabled: true` patches session Ingress host (only if you route via ingress-nginx instead of Apache).

Manual ingress patch (legacy troubleshooting only):

```bash
kubectl -n openvre patch ingress openvre-ix-<session-id> --type=merge --patch-file /tmp/ix-ing-patch.yaml
# patch file must set spec.rules[0].host to your domain.host
```

Ensure `scheduler.externalBase` matches how users open the portal (`http://openvre.local`).

## Will it work with only new scheduler + old frontend image?

**No.** You need **both**:

- `scheduler-3.1` (interactive session API)
- Frontend image built from **`openvre-dev-kubernetes-interactive-rstudio-live`** (PHP bridge)

Using `ghcr.io/inab/kubernetes-openvre-frontend:1.0` alone will not create interactive sessions correctly.
