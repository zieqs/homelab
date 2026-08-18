# zieqs/homelab

A GitOps-driven Kubernetes homelab. Every service lives in this repo as a Kubernetes manifest, and [Flux CD](https://fluxcd.io/) keeps the cluster in sync automatically — no SSHing in to hand-edit files.

## What's Running

| Service | Namespace | What it does |
|---|---|---|
| **[Pi-hole](https://pi-hole.net/)** | `networking` | DNS ad-blocking for the LAN, also exposed over Tailscale. |
| **[Tailscale Operator](https://tailscale.com/kubernetes-operator)** | `tailscale` | Mesh VPN + subnet router for `192.168.0.0/24`. Exposes services via `tailscale.com/expose`. |
| **[Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/)** | `networking` | Cloudflare-backed ingress for external access. |
| **Radar** | `radar` | Cluster dashboard / observability UI. |
| **[Glances](https://nicolargo.github.io/glances/)** | `monitoring` | System monitoring — a master + worker instance, one per node. |
| **[Homarr](https://homarr.dev/)** | `monitoring` | Dashboard / startpage. |
| **[Home Assistant](https://www.home-assistant.io/)** | `home-assistant` | Home automation (master node). |
| **[Samba](https://www.samba.org/)** | `storage` | File sharing for HDD1/HDD2, exposed over Tailscale. |
| **[Jellyfin](https://jellyfin.org/)** | `media` | Media server with GPU passthrough on the worker node. |
| **Radarr / Sonarr / Jellyseerr / Bazarr** | `media` | Movie/TV management, request and subtitle tooling (on the master node). |
| **qBittorrent / Prowlarr** | `media` | Torrent client (worker node), indexer manager (master node). |
| **[CouchDB](https://couchdb.apache.org/)** 3.5.2 | `obsidian` | [Obsidian Livesync](https://github.com/vrtmrz/obsidian-livesync) backend. 10Gi PVC. |
| **Minecraft** | `games` | NeoForge 1.21.1 server on the worker node (8G RAM), exposed via a [playit.gg](https://playit.gg) tunnel. |
| **[Vaultwarden](https://github.com/dani-garcia/vaultwarden)** | `vaultwarden` | Bitwarden-compatible password manager. 10Gi PVC. |

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  GitHub                          │
│   github.com/zieqs/homelab                      │
│   ├── apps/           (app manifests)           │
│   ├── infrastructure/ (tailscale, cloudflared,   │
│   │                    radar)                   │
│   └── clusters/       (Flux Kustomizations)     │
└──────────────┬──────────────────────────────────┘
               │ git poll (every 1m)
               ▼
┌─────────────────────────────────────────────────┐
│          Flux CD (v2.9.2)                        │
│  source-controller → kustomize-controller       │
│                 ↓                               │
│  sops decryption → helm-controller             │
└──────────────┬──────────────────────────────────┘
               │ reconcile
               ▼
┌─────────────────────────────────────────────────┐
│         Kubernetes Cluster (k3s, 2 nodes)        │
│                                                  │
│  ┌─ homelab (master) ────────┐  ┌─ homelab-worker1│
│  │  Pi-hole      │  Prowlarr │  │  Jellyfin (GPU) │
│  │  Glances      │  CouchDB  │  │  qBittorrent    │
│  │  Homarr       │  Samba    │  │  Glances-worker │
│  │  Home Assistant│  *arr    │  │  Minecraft      │
│  │  Vaultwarden  │  Radar    │  │                 │
│  └───────────────────────────┘  └─────────────────┘
│                                                  │
│  Tailscale (mesh + subnet route 192.168.0.0/24) │
│  Cloudflare Tunnel (external ingress)            │
│  playit.gg (Minecraft tunnel)                    │
└─────────────────────────────────────────────────┘
```

## Directory Layout

```
homelab/
├── .sops.yaml              # SOPS config (age key, encrypted regex)
├── .gitignore              # Ignores age.agekey and plaintext secrets/
├── apps/                   # Application manifests
│   ├── pihole/             # Helm release + ingress + encrypted secret
│   ├── glances/            # Glances master + worker
│   ├── homarr/             # Dashboard + encrypted secret
│   ├── home-assistant/     # HA deployment + storage + ingress
│   ├── media/              # Jellyfin, *arr, qBittorrent, Prowlarr
│   ├── minecraft/          # NeoForge server + playit.gg tunnel + secrets
│   ├── obsidian/           # CouchDB + encrypted secret
│   ├── samba/              # File sharing + encrypted secret
│   └── vaultwarden/        # Helm release + PVC + ingress
├── infrastructure/
│   ├── tailscale/          # Tailscale operator Helm release
│   ├── cloudflared/        # Cloudflare tunnel + encrypted secret
│   └── radar/              # Radar Helm release
└── clusters/homelab/       # Flux Kustomizations (source of truth)
    ├── flux-system/        # Bootstrapped Flux components (don't edit)
    ├── app-*.yaml          # One Kustomization per app
    └── infra-*.yaml        # tailscale, cloudflared, radar
```

## Secret Management

Secrets are encrypted with [SOPS](https://github.com/getsops/sops) using an age key and stored as `secrets/*.enc.yaml` inside each app's directory (e.g. `pihole-secret`, `homarr-secret`, `couchdb-secret`, `cloudflared-secret`, `samba-secret`, `minecraft-secret`). Plaintext `secrets/*.yaml` files are gitignored — only the encrypted versions are ever committed.

Each Flux `Kustomization` that consumes secrets has a `decryption` block pointing at the age key:

```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

The `sops-age` Secret holds your age private key and must exist in the `flux-system` namespace:

```bash
kubectl create secret generic sops-age \
  --namespace flux-system \
  --from-file=age.agekey=<path-to-your-age-key>
```

## Bootstrap From Scratch

1. **Clone the repo**

   ```bash
   git clone https://github.com/zieqs/homelab.git
   cd homelab
   ```

2. **Bootstrap Flux**

   ```bash
   flux bootstrap github \
     --owner=YOUR_GITHUB_USER \
     --repository=homelab \
     --branch=main \
     --path=clusters/homelab
   ```

3. **Import the age key**

   ```bash
   kubectl create secret generic sops-age \
     --namespace flux-system \
     --from-file=age.agekey=<path-to-your-age-key>
   ```

4. **Create any remaining secrets** not yet migrated to SOPS (e.g. Tailscale OAuth credentials):

   ```bash
   kubectl create secret generic operator-oauth \
     --namespace tailscale \
     --from-literal=client_id=<your-client-id> \
     --from-literal=client_secret=<your-client-secret>
   ```

5. **Let Flux reconcile**

   ```bash
   flux get kustomizations --watch
   ```

## Highlights

- **Pi-hole** — community Helm chart, LAN-wide DNS via a LoadBalancer service, password injected from a SOPS secret, 5Gi `local-path` storage.
- **Media stack** — Jellyfin with `/dev/dri` GPU passthrough on the worker, the *arr suite + Prowlarr on the master (hostPath at `/mnt/HDD2/media-stack`), qBittorrent downloads on the worker. All behind Traefik ingresses.
- **Home Assistant** — pinned to the master node, 2Gi PVC, `Recreate` strategy.
- **Samba** — hostNetwork on the master sharing HDD1/HDD2, exposed over Tailscale.
- **Minecraft** — NeoForge 1.21.1 (`itzg/minecraft-server:java21`), 8G RAM on the worker, storage at `/mnt/sata-storage/minecraft-data`, external access via a playit.gg sidecar. Currently scaled to 0.
- **Radar** — Helm-installed cluster dashboard/observability UI.
- **Glances** — master + worker deployments with `hostPID` and Docker socket access for full system visibility.
- **Tailscale** — operator with subnet-route advertising for `192.168.0.0/24`; annotate any Service with `tailscale.com/expose: "true"` to get a mesh hostname.
- **Cloudflare Tunnel** — `cloudflared` pinned to the master, token stored in a SOPS-encrypted secret.

## How It Stays Healthy

- **GitOps by default** — the desired state lives in Git; Flux reconciles every few minutes and prunes anything it no longer manages.
- **Node pinning** — workloads are scheduled to the node that holds their data or GPU (`kubernetes.io/hostname`), since storage is node-local (`hostPath` / `local-path`).
- **Renovate** keeps image tags and Helm chart versions up to date, with digest-pinning for Docker updates.
- **CI** (`.github/workflows/manifests.yaml`) validates every push to `main`: builds each overlay with Kustomize, strips SOPS stanzas, and checks manifests with `kubeconform`, then verifies every `*.enc.yaml` can be decrypted.

## Sync Intervals

| Resource | Interval |
|---|---|
| GitRepository (Git poll) | 1 minute |
| Root Kustomization | 10 minutes |
| Pi-hole Kustomization | 60 seconds |
| App Kustomizations (everything else) | 5 minutes |
| Infrastructure Kustomizations | 10 minutes |
| HelmReleases (chart updates) | 1–2 hours |

## Adding a New App

1. Create manifests in `apps/<app-name>/` (namespace, deployment, service, ingress, …).
2. Encrypt any secrets with SOPS and store them as `apps/<app-name>/secrets/<name>.enc.yaml`.
3. List everything in a `kustomization.yaml`.
4. Add a Flux `Kustomization` in `clusters/homelab/app-<app-name>.yaml` (with a `decryption` block if it has secrets).
5. Commit and push — Flux picks it up automatically.

## License

MIT
