# Immich Deployment on Homelab

**Date:** 2026-07-29
**Status:** Approved

## Overview

Deploy [Immich](https://immich.app/) — a self-hosted photo and video management solution — to the homelab k3s cluster. Managed via Flux CD, with media storage on master node hostPath, Traefik ingress, and SOPS-encrypted secrets.

## Approach

Official Immich Helm chart (from `https://github.com/immich-app/immich-charts`) with bundled PostgreSQL and Redis subcharts. Follows the same pattern as pihole/vaultwarden (HelmRepository + HelmRelease + Kustomize overlay).

## Structure

```
apps/immich/
├── kustomization.yaml
├── namespace.yaml
├── repository.yaml
├── release.yaml
├── storage.yaml
└── secrets/
    └── immich-secret.enc.yaml
```

Flux Kustomization at `clusters/homelab/app-immich.yaml`.

## Storage

| Purpose | Type | Path/Class | Size | Notes |
|---------|------|------------|------|-------|
| Photo/video library | hostPath PV → PVC | `/mnt/HDD2/immich` | full disk | Binds to `homelab` node, `Retain` reclaim |
| PostgreSQL | `local-path` PVC | `storageClass: local-path` | 10Gi | Managed by bundled subchart |
| Redis | `local-path` PVC | `storageClass: local-path` | 1Gi | Managed by bundled subchart |

## Networking

- **Ingress:** `immich.zieqs.online` via Traefik (ingressClassName: `traefik`, HTTP only, Cloudflare terminates TLS)
- **Node:** Pinned to master node (`homelab`) via `nodeSelector`

## Configuration

- Image tags pinned to a specific Immich release
- ML service enabled (CPU mode)
- Node selector: `kubernetes.io/hostname: homelab`
- Bundled PostgreSQL and Redis with `local-path` storageClass
- Resource requests/limits set for each component

## Secrets

Explicit DB passwords in SOPS-encrypted Secret (`secrets/immich-secret.enc.yaml`):
- `db-password`
- `db-user`
- Redis password

## Flux Integration

New `Kustomization` resource in `flux-system` namespace:
- Path: `./apps/immich`
- Interval: 5m
- SOPS decryption enabled
- Prune: true
