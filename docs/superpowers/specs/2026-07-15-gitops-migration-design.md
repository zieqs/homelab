# GitOps Migration Design

**Date:** 2026-07-15
**Status:** Draft
**Goal:** Migrate all manually deployed apps/services on the homelab Kubernetes cluster to GitOps-managed deployments using Flux CD v2.

---

## 1. Current State

A 2-node k3s cluster (one control-plane `homelab`, one worker `homelab-worker1`) running ~11 user apps across 5 namespaces, all deployed via `kubectl apply -f` with raw YAMLs. Flux CD v2.9.2 is bootstrapped but managing nothing.

| Namespace | Apps |
|-----------|------|
| `default` | homarr (dashboard), pihole (DNS/ad-blocker) |
| `games` | rlcraft-server (Minecraft) |
| `media` | bazarr, jellyfin, jellyseerr, prowlarr, qbittorrent, radarr, sonarr |
| `obsidian` | couchdb |
| `tailscale` | tailscale-operator, pihole-dns sidecar |

Infrastructure: traefik (ingress), cloudflared (tunnel), coredns, local-path-provisioner, metrics-server, glances (monitoring).

---

## 2. Target State

```
clusters/homelab/
├── kustomization.yaml            # Root Kustomization — includes all groups below
├── flux-system/                  # Existing Flux manifests (untouched)
├── config/
│   └── namespaces.yaml           # All app namespaces defined here
├── dashboard/
│   ├── kustomization.yaml
│   └── homarr/
│       ├── helmrelease.yaml
│       └── pvc.yaml
├── networking/
│   ├── kustomization.yaml
│   ├── pihole/
│   │   └── helmrelease.yaml
│   └── cloudflared/
│       └── helmrelease.yaml
├── media/
│   ├── kustomization.yaml
│   ├── jellyfin/
│   │   ├── helmrelease.yaml
│   │   └── secrets/
│   │       └── config.sops.yaml
│   ├── radarr/
│   │   ├── helmrelease.yaml
│   │   └── secrets/
│   │       └── config.sops.yaml
│   ├── sonarr/
│   │   ├── helmrelease.yaml
│   │   └── secrets/
│   │       └── config.sops.yaml
│   ├── prowlarr/
│   │   ├── helmrelease.yaml
│   │   └── secrets/
│   │       └── config.sops.yaml
│   ├── bazarr/
│   │   ├── helmrelease.yaml
│   │   └── secrets/
│   │       └── config.sops.yaml
│   ├── qbittorrent/
│   │   ├── helmrelease.yaml
│   │   └── secrets/
│   │       └── config.sops.yaml
│   └── jellyseerr/
│       ├── helmrelease.yaml
│       └── secrets/
│           └── config.sops.yaml
├── games/
│   ├── kustomization.yaml
│   └── rlcraft/
│       ├── deployment.yaml
│       └── service.yaml
├── obsidian/
│   ├── kustomization.yaml
│   └── couchdb/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── pvc.yaml
├── observability/
│   ├── kustomization.yaml
│   └── glances/
│       ├── daemonset.yaml
│       └── service.yaml
└── tailscale/
    ├── kustomization.yaml
    └── operator/
        └── helmrelease.yaml
```

### Namespace Plan

| Namespace | Contains | Ingress Domain(s) |
|-----------|----------|-------------------|
| `dashboard` | homarr | homarr.zieqs.online |
| `networking` | pihole, cloudflared | pihole.zieqs.online, *.zieqs.online |
| `media` | radarr, sonarr, prowlarr, bazarr, qbittorrent, jellyfin, jellyseerr | radarr.zieqs.online, sonarr.zieqs.online, seerr.zieqs.online, qbit.zieqs.online, prowlarr.zieqs.online, jellyfin.zieqs.online |
| `games` | rlcraft-server | (none — NodePort) |
| `obsidian` | couchdb | couchdb.zieqs.online |
| `observability` | glances | glancesmaster.zieqs.online, glancesworkerone.zieqs.online |
| `tailscale` | tailscale-operator | (none) |

---

## 3. Technology Choices

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Deployment method | **HelmReleases** for apps with charts, **raw Kustomize** for others | Most apps have maintained Helm charts; raw YAML for simple apps avoids chart overhead |
| Secrets management | **SOPS + Age** | Built into Flux, no extra operators, simple for homelab scale |
| Declarative approach | **Kustomize overlays** via Flux | Already bootstrapped, no extra CRDs needed |
| Storage | **local-path-provisioner** (existing) | PVs already bound; PVCs kept as-is in HelmRelease values |

### Helm Chart Sources

Most apps will use community charts from **bjw-s** (bjw-s-libs/app-template) or **truecharts** — to be determined per app during migration based on current deployment patterns.

---

## 4. Secrets Strategy (SOPS + Age)

### Setup (one-time)
1. Generate Age keypair: `age-keygen -o age-key.txt`
2. Create Flux-compatible Secret in `flux-system` namespace from the private key
3. Configure `.sops.yaml` in repo root with the Age public key

### Per-app workflow
1. Write `config.yaml` (plaintext) with the app's secrets
2. `sops --encrypt config.yaml > config.sops.yaml`
3. Commit only `config.sops.yaml` to Git
4. `HelmRelease` references the SOPS-encrypted file as a values source
5. Flux decrypts at sync-time using the Age key stored in-cluster

### What gets encrypted
- API keys (radarr, sonarr, prowlarr, lidarr, readarr, bazarr)
- qbittorrent credentials
- pihole admin password
- Jellyfin admin config (if not default)
- Cloudflare tunnel token

---

## 5. Migration Order (9 Phases)

| Phase | App | Method | Complexity | Notes |
|-------|-----|--------|------------|-------|
| 1 | couchdb (obsidian) | Raw Kustomize | Low | No secrets, single pod + PVC, build confidence |
| 2 | rlcraft-server (games) | Raw Kustomize | Low | No ingress, NodePort only |
| 3 | jellyfin (media) | HelmRelease | Medium | First HelmRelease, test the Flux Helm pattern |
| 4 | radarr, sonarr, prowlarr, bazarr, qbittorrent, jellyseerr (media) | HelmRelease | Medium | Same chart pattern, each has API secret keys |
| 5 | homarr (dashboard) | HelmRelease | Low | PVC + ingress, no secrets, straightforward |
| 6 | pihole (networking) | HelmRelease | Medium | DNS config, secrets, Tailscale integration |
| 7 | cloudflared (networking) | HelmRelease | Low | Simple config |
| 8 | glances (observability) | Raw Kustomize | Low | DaemonSet, optional |
| 9 | tailscale-operator (tailscale) | HelmRelease | Low | Already Helm-managed, adopt via Flux |

### Migration Workflow (per app)

```
1. EXTRACT   → kubectl get deploy,svc,ingress,pvc,cm,secret -n <ns> -o yaml
2. STRIP     → Remove status, managedFields, resourceVersion, uid, ownerReferences
3. CONVERT   → Rewrite as HelmRelease or clean Kustomize resource
4. ENCRYPT   → sops --encrypt any secrets
5. COMMIT    → git add && git commit && git push
6. VERIFY    → kubectl get pods -n <ns> -w  →  pod Running  →  app accessible
7. CLEANUP   → Delete old manual YAMLs, remove old kubectl-apply'd manifests
```

---

## 6. HelmRelease Template

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <app>
  namespace: <namespace>
spec:
  interval: 15m
  chart:
    spec:
      chart: <chart-name>
      sourceRef:
        kind: HelmRepository
        name: <repo-name>
        namespace: flux-system
      interval: 1h
  values:
    # Ingress
    ingress:
      main:
        enabled: true
        ingressClassName: traefik
        hosts:
          - host: <app>.zieqs.online
            paths:
              - path: /
                pathType: Prefix
    # Persistence
    persistence:
      config:
        enabled: true
        existingClaim: <existing-pvc>  # Use existing PVC to preserve data
    # Secrets (via SOPS)
    # config.sops.yaml mounts encrypted values
```

---

## 7. Rollback Strategy

Each app's old `kubectl apply` YAMLs should be **saved but not applied** during migration. If a Flux-managed app fails:

1. Flux will retry on its own (default backoff)
2. If broken beyond recovery, delete the HelmRelease/Kustomize resource from Git and commit
3. Re-apply the old YAML manually via `kubectl apply -f`
4. Debug the Flux manifest at leisure

No app should experience downtime since Flux creates the exact same resource shapes as the manual YAMLs (same service names, same PVC names, same selectors). Pods migrate transparently.

---

## 8. Acceptance Criteria

- [ ] All apps running in their respective namespaces
- [ ] All ingresses functional at their expected domains
- [ ] All existing PVCs preserved and data intact
- [ ] Secrets encrypted via SOPS, no plaintext secrets in Git
- [ ] Flux reconciles automatically with no errors
- [ ] `flux get all` shows no failures
- [ ] Old `kubectl apply` YAMLs are removed from active use
- [ ] Rollback procedure documented in case of issues
