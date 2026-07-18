# zieqs/homelab

A GitOps-driven Kubernetes homelab. Everything lives in this repo, and [Flux CD](https://fluxcd.io/) keeps the cluster in sync — no SSHing in to tweak manifests by hand.

## What's Running

| Service | Namespace | What it does |
|---|---|---|
| **[Pi-hole](https://pi-hole.net/)** | `networking` | DNS ad-blocking for the LAN. Web UI at `pihole.zieqs.online`. DNS also exposed over Tailscale at `cluster-pihole-dns`. |
| **[Tailscale Operator](https://tailscale.com/kubernetes-operator)** | `tailscale` | Mesh VPN + subnet router for `192.168.0.0/24`. Exposes services via `tailscale.com/expose` annotations. |
| **[Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/)** | `kube-system` | Cloudflare-backed ingress tunnel for external access. Token encrypted with SOPS. |
| **[Glances](https://nicolargo.github.io/glances/)** | `kube-system` | System monitoring — master at `glancesmaster.zieqs.online`, worker at `glancesworkerone.zieqs.online`. |
| **[Homarr](https://homarr.dev/)** | `monitoring` | Dashboard / startpage at `homarr.zieqs.online`. Encryption key stored in a SOPS-encrypted secret. |
| **[Jellyfin](https://jellyfin.org/)** | `media` | Media server at `jellyfin.zieqs.online` with GPU passthrough on the worker node. |
| **Radarr / Sonarr / Seerr / Bazarr** | `media` | Radarr at `radarr.zieqs.online`, Sonarr at `sonarr.zieqs.online` (scaled to 0), Seerr at `seerr.zieqs.online`, Bazarr at `bazarr.zieqs.online`. |
| **qBittorrent / Prowlarr** | `media` | Torrent client at `qbit.zieqs.online` (worker node), indexer manager at `prowlarr.zieqs.online` (master node). |
| **[CouchDB](https://couchdb.apache.org/)** 3.4.2 | `obsidian` | [Obsidian Livesync](https://github.com/vrtmrz/obsidian-livesync) backend at `couchdb.zieqs.online`. 10Gi PVC, credentials in a SOPS-encrypted secret. |
| **RLCraft** (Minecraft) | `games` | Forge 1.12.2 modded server on the worker node (8G RAM). Exposed via [playit.gg](https://playit.gg) tunnel sidecar. Secret encrypted with SOPS. |
| **[Vaultwarden](https://github.com/dani-garcia/vaultwarden)** | `vaultwarden` | Bitwarden-compatible password manager. Helm chart from `johanneskastl`. Ingress at `vaultwarden.zieqs.online`. 10Gi PVC. |

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  GitHub                          │
│   github.com/zieqs/homelab                      │
│   ├── apps/     (pihole, glances, homarr,       │
│   │              media, obsidian, minecraft,     │
│   │              vaultwarden)                    │
│   ├── infrastructure/ (tailscale, cloudflared)   │
│   └── clusters/ (Flux Kustomizations + SOPS)    │
└──────────────┬──────────────────────────────────┘
               │ git poll (every 1m)
               ▼
┌─────────────────────────────────────────────────┐
│          Flux CD (v2.9.2)                        │
│  source-controller → kustomize-controller       │
│  → helm-controller                              │
│  → sops decryption                              │
└──────────────┬──────────────────────────────────┘
               │ reconcile
               ▼
┌─────────────────────────────────────────────────┐
│         Kubernetes Cluster (k3s, 2 nodes)        │
│                                                  │
│  ┌─ homelab (master) ──────┐  ┌─ homelab-worker1 │
│  │  Pi-hole     │  Prowlarr │  │  Jellyfin (GPU)  │
│  │  Glances     │  CouchDB  │  │  qBittorrent     │
│  │  Radarr      │  Homarr   │  │  Glances-worker  │
│  │  Sonarr      │  Bazarr   │  │  RLCraft (MC)     │
│  │  Seerr       │  Vaultwarden│  │                   │
│  └──────────────────────────┘  └───────────────────┘
│                                                  │
│  Tailscale (mesh + subnet route 192.168.0.0/24) │
│  Cloudflare Tunnel (external ingress)            │
│  playit.gg (Minecraft tunnel)                    │
└─────────────────────────────────────────────────┘
```

## Directory Layout

```
homelab/
├── .gitignore                # Ignores age.key and unencrypted secrets/
├── .sops.yaml                # SOPS config (age key, encrypted regex)
├── apps/                     # Application manifests
│   ├── pihole/               # HelmRelease + Ingress + encrypted secret
│   ├── glances/              # Glances master + worker deployments
│   ├── homarr/               # Homarr dashboard + encrypted secret
│   ├── media/                # Full media stack (Jellyfin, *arr, qBittorrent)
│   ├── obsidian/             # CouchDB + encrypted secret
│   ├── minecraft/            # RLCraft server + playit.gg tunnel + encrypted secret
│   └── vaultwarden/          # Vaultwarden HelmRelease + PVC + ingress
├── clusters/
│   └── homelab/              # Flux Kustomizations (each with SOPS decryption)
│       ├── flux-system/      # Bootstrapped Flux components
│       ├── app-pihole.yaml
│       ├── app-glaces.yaml
│       ├── app-homarr.yaml
│       ├── app-media.yaml
│       ├── app-obsidian.yaml
│       ├── app-minecraft.yaml
│       ├── app-vaultwarden.yaml
│       ├── infra-tailscale.yaml
│       └── infra-cloudflared.yaml
└── infrastructure/
    ├── tailscale/            # Tailscale operator HelmRelease
    └── cloudflared/          # Cloudflare tunnel + encrypted secret
```

## Secret Management

Secrets are encrypted with [SOPS](https://github.com/getsops/sops) using an age key and stored in `secrets/*.enc.yaml` files within each app's directory. The `.gitignore` ensures plain secret files (`secrets/*.yaml`) are never committed.

Each Flux `Kustomization` that consumes encrypted secrets has a `decryption` block:

```yaml
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

The `sops-age` Kubernetes Secret must contain your age private key. Create it with:

```bash
kubectl create secret generic sops-age \
  --namespace flux-system \
  --from-file=age.agekey=<path-to-your-age-key>
```

### Currently encrypted with SOPS

| App | Secret | Contents |
|---|---|---|
| Pi-hole | `pihole-secret` | Admin password (injected via Helm `valuesFrom`) |
| Homarr | `homarr-secret` | `SECRET_ENCRYPTION_KEY` |
| CouchDB | `couchdb-secret` | `COUCHDB_USER` + `COUCHDB_PASSWORD` |
| Cloudflare Tunnel | `cloudflared-secret` | Tunnel token |
| RLCraft | `rlcraft-secret` | playit.gg tunnel `SECRET_KEY` |

## Prerequisites

- A Kubernetes cluster (I run [k3s](https://k3s.io/) with two nodes: `homelab` and `homelab-worker1`)
- `kubectl` configured for your cluster
- `flux` CLI v2.9+
- [SOPS](https://github.com/getsops/sops) + an [age](https://github.com/FiloSottile/age) key pair
- A GitHub [deploy key](https://fluxcd.io/flux/installation/bootstrap/github/)

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

4. **Create any remaining secrets** not yet migrated to SOPS (e.g., Tailscale OAuth credentials)

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

## Under the Hood

### Pi-hole (`apps/pihole/`)

- **Chart:** `mojo2600/pihole` community Helm chart
- **DNS:** `LoadBalancer` service for LAN-wide DNS
- **Web UI:** Traefik Ingress at `pihole.zieqs.online`, password injected via `spec.valuesFrom` from a SOPS-encrypted secret
- **Storage:** 5Gi `local-path` PVC
- **Tailscale:** `ts-pihole-dns` service exposed over Tailscale at `cluster-pihole-dns`

### Tailscale Operator (`infrastructure/tailscale/`)

- **Subnet router:** Advertises `192.168.0.0/24` for LAN access from the mesh
- **Service exposure:** Annotate any Service with `tailscale.com/expose: "true"` to get a Tailscale hostname
- **Auth:** `operator-oauth` secret in the `tailscale` namespace

### Cloudflare Tunnel (`infrastructure/cloudflared/`)

- A `cloudflared` deployment pinned to the master node, using a tunnel token stored in a SOPS-encrypted secret
- Handles external ingress for services routed through Cloudflare

### Glances (`apps/glances/`)

- Two deployments — `glances-master` on `homelab` and `glances-worker` on `homelab-worker1`
- Both run with `hostPID: true` and Docker socket access for full system visibility
- Separate ingresses: `glancesmaster.zieqs.online` and `glancesworkerone.zieqs.online`

### Homarr (`apps/homarr/`)

- Single deployment on the master node with a 2Gi PVC
- `SECRET_ENCRYPTION_KEY` sourced from a SOPS-encrypted secret
- Ingress at `homarr.zieqs.online`

### Media Stack (`apps/media/`)

- **Jellyfin** — GPU-accelerated (`/dev/dri` passthrough) on the worker node, NFS-backed media storage
- **Radarr / Sonarr / Seerr / Bazarr** — On the master node, hostPath at `/mnt/HDD2/media-stack`. Sonarr scaled to 0.
- **qBittorrent** — Worker node, downloads on NFS mount
- **Prowlarr** — Master node for indexer management
- All services have Traefik Ingresses on `*.zieqs.online`

### CouchDB (`apps/obsidian/`)

- CouchDB 3.4.2 for [Obsidian Livesync](https://github.com/vrtmrz/obsidian-livesync)
- 10Gi PVC, credentials in a SOPS-encrypted secret
- Ingress at `couchdb.zieqs.online`

### RLCraft Minecraft (`apps/minecraft/`)

- **Image:** `itzg/minecraft-server:java8-multiarch` running Forge 1.12.2
- **Node:** Pinned to `homelab-worker1` with 8G RAM allocation
- **Storage:** HostPath at `/mnt/sata-storage/minecraft-data`
- **Tunnel:** A [playit.gg](https://playit.gg) sidecar container exposes the server externally. Its secret key is stored in a SOPS-encrypted secret.
- **Service:** `ClusterIP` (playit handles external traffic)

### Vaultwarden (`apps/vaultwarden/`)

- **Chart:** `johanneskastl/vaultwarden` Helm chart (^6.0.0)
- **Storage:** 10Gi PVC for persistent data
- **Ingress:** Traefik at `vaultwarden.zieqs.online`
- **Secrets:** Flux Kustomization configured with SOPS decryption

### CI/CD

A GitHub Actions workflow (`.github/workflows/manifests.yaml`) runs on every push and PR to `main`:
- **validate** — builds every `apps/*/` and `infrastructure/*/` overlay with Kustomize, strips SOPS stanzas, and validates manifests with kubeconform
- **sops-check** — imports the `AGE_SECRET_KEY` repository secret and verifies all `*.enc.yaml` files can be decrypted

### Sync Intervals

| Resource | Interval |
|---|---|
| GitRepository (Git poll) | 1 minute |
| Root Kustomization | 10 minutes |
| Pi-hole Kustomization | 60 seconds |
| Media / Homarr / Obsidian / Glances / Minecraft / Vaultwarden Kustomizations | 5 minutes |
| Tailscale / Cloudflared Kustomizations | 10 minutes |
| HelmReleases (chart updates) | 1–2 hours |

## Adding a New App

1. Create manifests in `apps/<app-name>/` (namespace, deployments, services, ingress, etc.)
2. If the app needs secrets, encrypt them with SOPS and store in `apps/<app-name>/secrets/<name>.enc.yaml`
3. Bundle everything with `kustomization.yaml` (include the `.enc.yaml` files in `resources`)
4. Add a Flux `Kustomization` in `clusters/homelab/app-<app-name>.yaml` with a `decryption` block
5. Commit, push — Flux picks it up automatically

## License

MIT
