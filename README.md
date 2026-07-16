# zieqs/homelab

A GitOps-driven Kubernetes homelab. Everything here lives in this repo, and [Flux CD](https://fluxcd.io/) keeps the cluster in sync — no SSHing in to tweak manifests by hand.

## What's Running

| Service | Namespace | What it does |
|---|---|---|
| **[Pi-hole](https://pi-hole.net/)** | `networking` | DNS ad-blocking for the LAN. Web UI at `pihole.zieqs.online`. DNS also exposed over Tailscale for mesh devices at `cluster-pihole-dns`. |
| **[Tailscale Operator](https://tailscale.com/kubernetes-operator)** | `tailscale` | Connects the cluster to my Tailscale mesh. Acts as a subnet router for `192.168.0.0/24` and exposes services via `tailscale.com/expose` annotations. |
| **[Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/)** | `kube-system` | Cloudflare-backed ingress tunnel for services exposed to the internet. |
| **[Glances](https://nicolargo.github.io/glances/)** | `kube-system` | System monitoring — master node at `glancesmaster.zieqs.online`, worker node at `glancesworkerone.zieqs.online`. |
| **[Homarr](https://homarr.dev/)** | `monitoring` | Dashboard / startpage at `homarr.zieqs.online`. |
| **[Jellyfin](https://jellyfin.org/)** | `media` | Media server at `jellyfin.zieqs.online` with GPU passthrough on the worker node. |
| **Radarr / Sonarr / Jellyseerr / Bazarr** | `media` | The *arr suite — Radarr at `radarr.zieqs.online`, Sonarr at `sonarr.zieqs.online`, Jellyseerr at `seerr.zieqs.online`, Bazarr at `bazarr.zieqs.online`. Sonarr is currently scaled to 0. |
| **qBittorrent / Prowlarr** | `media` | Torrent client at `qbit.zieqs.online` (worker node) and indexer manager at `prowlarr.zieqs.online` (master node). |
| **[CouchDB](https://couchdb.apache.org/)** 3.4.2 | `obsidian` | [Obsidian Livesync](https://github.com/vrtmrz/obsidian-livesync) backend at `couchdb.zieqs.online`. 10Gi PVC for data. |

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  GitHub                          │
│   github.com/zieqs/homelab                      │
│   ├── apps/     (pihole, glances, homarr,       │
│   │              media, obsidian)                │
│   ├── infrastructure/ (tailscale, cloudflared)   │
│   └── clusters/ (Flux Kustomizations)           │
└──────────────┬──────────────────────────────────┘
               │ git poll (every 1m)
               ▼
┌─────────────────────────────────────────────────┐
│          Flux CD (v2.9.2)                        │
│  source-controller → kustomize-controller       │
│  → helm-controller                              │
└──────────────┬──────────────────────────────────┘
               │ reconcile
               ▼
┌─────────────────────────────────────────────────┐
│         Kubernetes Cluster (k3s)                 │
│                                                  │
│  ┌─ homelab (master) ──────┐  ┌─ homelab-worker1 │
│  │  Pi-hole     │  Prowlarr │  │  Jellyfin (GPU)  │
│  │  Glances     │  CouchDB  │  │  qBittorrent     │
│  │  Radarr      │  Homarr   │  │  Glances-worker  │
│  │  Sonarr      │  Bazarr   │  │                   │
│  │  Jellyseerr  │           │  │                   │
│  └──────────────────────────┘  └───────────────────┘
│                                                  │
│  Tailscale (mesh + subnet route 192.168.0.0/24) │
│  Cloudflare Tunnel (external ingress)            │
└─────────────────────────────────────────────────┘
```

## Directory Layout

```
homelab/
├── .sops.yaml               # SOPS config for encrypted secrets
├── apps/                    # Application manifests
│   ├── pihole/              # Pi-hole HelmRelease + Ingress
│   ├── glances/             # Glances deployments (master + worker)
│   ├── homarr/              # Homarr dashboard
│   ├── media/               # Full media stack (Jellyfin, *arr, qBittorrent)
│   └── obsidian/            # CouchDB for Obsidian Livesync
├── clusters/
│   └── homelab/             # Flux Kustomizations wiring apps + infra
│       ├── flux-system/     # Bootstrapped Flux components
│       ├── app-pihole.yaml
│       ├── app-glaces.yaml
│       ├── app-homarr.yaml
│       ├── app-media.yaml
│       ├── app-obsidian.yaml
│       ├── infra-tailscale.yaml
│       └── infra-cloudflared.yaml
└── infrastructure/          # Shared infrastructure
    ├── tailscale/            # Tailscale operator
    └── cloudflared/          # Cloudflare tunnel
```

Standard GitOps layout — `clusters/homelab/` contains Flux `Kustomization` resources that point at `apps/` and `infrastructure/` directories. Add a new app by creating manifests under `apps/` and wiring it in with a new Kustomization.

## Prerequisites

- A Kubernetes cluster (I run [k3s](https://k3s.io/) with two nodes: `homelab` and `homelab-worker1`)
- `kubectl` configured to point at your cluster
- `flux` CLI v2.9+
- A GitHub account and a [deploy key](https://fluxcd.io/flux/installation/bootstrap/github/) with write access to this repo
- An [age key](https://github.com/FiloSottile/age) for SOPS-encrypted secrets

## Bootstrap From Scratch

If you want to stand up your own version of this setup:

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

   This installs Flux and creates the root GitRepository + Kustomization pointing at `clusters/homelab/`.

3. **Configure secrets**

   ```bash
   # Pi-hole admin password
   kubectl create secret generic pihole-password \
     --namespace networking \
     --from-literal=password=<your-password>

   # Tailscale OAuth credentials
   kubectl create secret generic operator-oauth \
     --namespace tailscale \
     --from-literal=client_id=<your-client-id> \
     --from-literal=client_secret=<your-client-secret>

   # SOPS age key (if using encrypted secrets)
   kubectl create secret generic sops-age \
     --namespace flux-system \
     --from-file=age.agekey=<path-to-your-age-key>
   ```

4. **Let Flux do its thing**

   ```bash
   flux get kustomizations --watch
   ```

   Within a minute, the cluster should be running everything.

## Under the Hood

### Pi-hole (`apps/pihole/`)

- **Chart:** `mojo2600/pihole` community Helm chart
- **DNS:** `LoadBalancer` service for LAN DNS resolution
- **Web UI:** Traefik Ingress at `pihole.zieqs.online` with password auth
- **Storage:** 5Gi `local-path` PVC
- **Secrets:** SOPS-encrypted for the admin password
- **Tailscale:** A second service (`ts-pihole-dns`) exposes DNS over Tailscale at `cluster-pihole-dns`

### Tailscale Operator (`infrastructure/tailscale/`)

- **Subnet router:** Advertises `192.168.0.0/24` so mesh devices can reach the home LAN
- **Service exposure:** Any Service annotated with `tailscale.com/expose: "true"` gets a Tailscale hostname
- **Auth:** `operator-oauth` secret in the `tailscale` namespace

### Cloudflare Tunnel (`infrastructure/cloudflared/`)

- A single `cloudflared` deployment in `kube-system`, pinned to the master node, running with a tunnel token. This handles external ingress for services exposed via Cloudflare.

### Glances (`apps/glances/`)

- Two deployments — `glances-master` on `homelab` and `glances-worker` on `homelab-worker1`
- Both run `nicolargo/glances:latest-full` with `hostPID: true` and Docker socket access for full system visibility
- Separate ingresses for each node

### Homarr (`apps/homarr/`)

- Single deployment on the master node with a 2Gi PVC for app data
- Ingress at `homarr.zieqs.online`

### Media Stack (`apps/media/`)

- **Jellyfin** — GPU-accelerated (passthrough of `/dev/dri`) on the worker node, NFS-backed storage
- **Radarr / Sonarr / Jellyseerr / Bazarr** — On the master node, hostPath storage at `/mnt/HDD2/media-stack`. Sonarr is scaled to 0.
- **qBittorrent** — Worker node, NFS-backed downloads
- **Prowlarr** — Master node for indexer management
- All have Traefik Ingresses on `*.zieqs.online`

### CouchDB (`apps/obsidian/`)

- CouchDB 3.4.2 for [Obsidian Livesync](https://github.com/vrtmrz/obsidian-livesync)
- 10Gi PVC for data, ingress at `couchdb.zieqs.online`

### Sync Intervals

| Resource | Interval |
|---|---|
| GitRepository (Git poll) | 1 minute |
| Root Kustomization | 10 minutes |
| Pi-hole Kustomization | 60 seconds |
| Media / Homarr / Obsidian / Glances Kustomizations | 5 minutes |
| Tailscale / Cloudflared Kustomizations | 10 minutes |
| HelmReleases (chart updates) | 1–2 hours |

### Secret Management

SOPS is configured in `.sops.yaml` to encrypt `.enc.yaml` files using an age key. Currently, `app-pihole.yaml` is set up with `decryption.provider: sops` referencing the `sops-age` secret.

## Adding a New App

1. Create manifests in `apps/<app-name>/` (namespace, deployments, services, ingress, etc.)
2. Bundle them with `kustomization.yaml`
3. Add a Flux `Kustomization` in `clusters/homelab/app-<app-name>.yaml` pointing at `./apps/<app-name>`
4. Commit, push, and Flux picks it up on the next reconciliation

## License

MIT
