# App Manifest Structure Reformat

## Goal

Standardize every app directory in `apps/` and `infrastructure/` to use a consistent set of file names: `deployment.yaml`, `networking.yaml`, `storage.yaml`, `kustomization.yaml`, and (for Helm charts) `repository.yaml` + `release.yaml`.

## Current State

- Manifests are organized inconsistently — some apps combine Deployment + Service + Ingress + PVC in a single file, others split across files arbitrarily.
- File names vary (e.g. `glances.yaml`, `jellyfin.yaml`, `home-assistant.yaml`, `couchdb.yaml`).
- Two deployment patterns: raw K8s manifests (most apps) and HelmRelease (pihole, vaultwarden, radar, tailscale).

## Standard File Layout

Per app directory, files are created **on demand** (only if the app needs that resource type):

| File | Content | Required? |
|------|---------|-----------|
| `namespace.yaml` | Namespace resource | Always |
| `deployment.yaml` | Deployment(s), StatefulSet(s), DaemonSet(s) | If workload exists |
| `networking.yaml` | Service(s), Ingress(es), NetworkPolicy | If networking exists |
| `storage.yaml` | PVC(s), PV(s) | If storage exists |
| `repository.yaml` | HelmRepository | If Helm chart |
| `release.yaml` | HelmRelease | If Helm chart |
| `kustomization.yaml` | Kustomize config listing resources | Always |
| `secrets/*.enc.yaml` | SOPS-encrypted secrets | If secrets exist |
| `secrets/*.yaml` | Plaintext secret (gitignored) | If secrets exist |

## Per-App Mapping

### apps/glances
- `glances.yaml` (Deployment `glances-master`, `glances-worker` + Service `glances-master-svc`, `glances-worker-svc`) → **deployment.yaml** (Deployments only) + **networking.yaml** (Services only)
- `ingress.yaml` → **networking.yaml** (merged)

### apps/homarr
- `homarr.yaml` (PVC 2Gi + Deployment + Service `homarr-web-svc`) → **deployment.yaml** + **storage.yaml** + **networking.yaml**
- `ingress.yaml` → **networking.yaml** (merged)

### apps/home-assistant
- `home-assistant.yaml` (PVC 2Gi + Deployment + Service `ha-web-svc`) → **deployment.yaml** + **storage.yaml** + **networking.yaml**
- `ingress.yaml` → **networking.yaml** (merged)

### apps/media
- `jellyfin.yaml` (Deployment + Service + Ingress) → **deployment.yaml** + **networking.yaml**
- `arr-stack.yaml` (Deployments: Radarr, Sonarr, Jellyseerr, Bazarr + Services + Ingress) → **deployment.yaml** + **networking.yaml**
- `torrent-stack.yaml` (Deployments: qBittorrent, Prowlarr + Services + Ingress) → **deployment.yaml** + **networking.yaml**
- Result: single **deployment.yaml** with all 8 Deployments; single **networking.yaml** with all Services + Ingresses

### apps/minecraft
- `minecraft.yaml` (Deployment + NodePort Service) → **deployment.yaml** + **networking.yaml**

### apps/obsidian
- `couchdb.yaml` (PVC 10Gi + Deployment + Service `couchdb-svc`) → **deployment.yaml** + **storage.yaml** + **networking.yaml**
- `ingress.yaml` → **networking.yaml** (merged)

### apps/pihole (Helm)
- `release.yaml` → stays as **release.yaml**
- `ingress.yaml` → **networking.yaml**
- `repository.yaml` → stays

### apps/samba
- `samba.yaml` (Deployment + Service `samba-svc`) → **deployment.yaml** + **networking.yaml**

### apps/vaultwarden (Helm)
- `vaultwarden.yaml` (PVC 10Gi + HelmRelease) → **storage.yaml** (PVC only) + **release.yaml** (HelmRelease, extracted from vaultwarden.yaml)
- `repository.yaml` → stays

### infrastructure/cloudflared
- `cloudflared.yaml` (Deployment) → **deployment.yaml**

### infrastructure/radar (Helm)
- `radar.yaml` (HelmRelease) → **release.yaml** (renamed)
- `repository.yaml` → stays

### infrastructure/tailscale (Helm)
- `release.yaml` → stays as **release.yaml**
- `pihole-tailscale.yaml` (Service) → **networking.yaml**
- `repository.yaml` → stays

## What Stays the Same

- `namespace.yaml` — separate file in every app
- `secrets/` — subdirectory with `.enc.yaml` + `.yaml` pair pattern
- `kustomization.yaml` — updated only to reference new file names
- `clusters/homelab/*.yaml` — Flux Kustomizations reference app paths (unchanged)
- Resource content — no functional changes, only file reorganization

## Execution Strategy

1. Process each app/infra directory sequentially
2. For each:
   - Read all existing manifest files
   - Group resources by kind category (Deployment/StatefulSet → deployment.yaml, Service/Ingress → networking.yaml, PVC → storage.yaml)
   - Write new files with merged content
   - Update kustomization.yaml to reference new filenames
   - Remove old files
3. Verify no resource content changed after each conversion
4. Run `kustomize build` to validate the cluster-level Flux Kustomizations still work

## Verification

- `git diff --stat` shows only file renames and kustomization.yaml changes, no functional YAML changes
- `kustomize build clusters/homelab/` succeeds for each app path
