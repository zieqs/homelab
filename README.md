# zieqs/homelab

A GitOps-driven Kubernetes homelab. Everything here is managed by [Flux CD](https://fluxcd.io/) — the cluster's desired state lives in this repo, and Flux keeps the cluster in sync automatically.

## What's Running

| Service | Namespace | How it's deployed | What it does |
|---|---|---|---|
| **[Pi-hole](https://pi-hole.net/)** | `networking` | Helm chart via Flux | DNS ad-blocking for the whole LAN. Web UI at [pihole.zieqs.online](http://pihole.zieqs.online). DNS also exposed over Tailscale for mesh devices. |
| **[Tailscale Operator](https://tailscale.com/kubernetes-operator)** | `tailscale` | Helm chart via Flux | Connects the cluster to my Tailscale mesh. Acts as a subnet router for `192.168.0.0/24` and exposes cluster services via `tailscale.com/expose` annotations. |

## Architecture

```
┌─────────────────────────────────────────────────┐
│                  GitHub                          │
│   github.com/zieqs/homelab                      │
│   ├── apps/pihole/                              │
│   └── infrastructure/tailscale/                 │
└──────────────┬──────────────────────────────────┘
               │ git pull (every 1m)
               ▼
┌─────────────────────────────────────────────────┐
│          Flux CD (v2.9.2)                        │
│  source-controller → kustomize-controller       │
│  → helm-controller                              │
└──────────────┬──────────────────────────────────┘
               │ reconcile
               ▼
┌─────────────────────────────────────────────────┐
│            Kubernetes Cluster                    │
│  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  networking/  │  │  tailscale/              │  │
│  │  - Pi-hole    │  │  - tailscale-operator   │  │
│  │  - ts-pihole  │  │  - subnet router        │  │
│  └──────┬───────┘  └──────────────────────────┘  │
│         │                                       │
│         ├── LoadBalancer → LAN DNS              │
│         ├── Ingress (Traefik) → Web UI          │
│         └── Tailscale expose → Mesh DNS         │
└─────────────────────────────────────────────────┘
```

## Directory Layout

```
homelab/
├── apps/                  # Application manifests
│   └── pihole/            # Pi-hole HelmRelease, Ingress, etc.
├── clusters/
│   └── homelab/           # Cluster-specific Flux config
│       ├── flux-system/   # Flux bootstrap manifests (auto-generated)
│       ├── app-pihole.yaml
│       └── infra-tailscale.yaml
├── infrastructure/        # Shared infrastructure
│   └── tailscale/         # Tailscale operator HelmRelease + config
└── README.md
```

Standard GitOps layout — `clusters/` wires up `apps/` and `infrastructure/` via Flux `Kustomization` resources. To add a new app, you create manifests under `apps/` and reference them from `clusters/`.

## Prerequisites

- A Kubernetes cluster (I use [k3s](https://k3s.io/) with `local-path` storage provisioner)
- `kubectl` configured to point at your cluster
- `flux` CLI v2.9+
- A GitHub account and a [deploy key](https://fluxcd.io/flux/installation/bootstrap/github/) with write access to this repo

## Bootstrap From Scratch

If you want to stand up your own version of this setup:

1. **Fork or clone the repo**

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

   This creates the `flux-system` namespace, installs Flux components, and sets up the GitRepository + root Kustomization that points at `clusters/homelab/`.

3. **Configure secrets**

   Flux Kustomizations reference secrets that you need to create manually:

   ```bash
   # Pi-hole admin password
   kubectl create secret generic pihole-password \
     --namespace networking \
     --from-literal=password=<your-password>

   # Tailscale OAuth credentials (if using operator auth)
   kubectl create secret generic operator-oauth \
     --namespace tailscale \
     --from-literal=client_id=<your-client-id> \
     --from-literal=client_secret=<your-client-secret>
   ```

4. **Let Flux do its thing**

   Within a minute or two, Flux picks up the manifests and deploys everything:

   ```bash
   flux get kustomizations --watch
   ```

## Under the Hood

### Pi-hole (`apps/pihole/`)

- **Chart:** `mojo2600/pihole` (community Helm chart)
- **DNS:** Exposed as a `LoadBalancer` service — any device on the LAN can use the cluster node IP as its DNS server
- **Web UI:** Served through a Traefik Ingress at `pihole.zieqs.online` — useful for checking query logs and whitelisting domains
- **Storage:** 5Gi PVC backed by `local-path` for persistent configuration across Pod restarts
- **Tailscale exposure:** A second service (`ts-pihole-dns`) exposes DNS port 53 over Tailscale with the hostname `cluster-pihole-dns`, so devices on my mesh network can use Pi-hole too

### Tailscale Operator (`infrastructure/tailscale/`)

- **Subnet router:** Advertises `192.168.0.0/24` so Tailscale-connected devices can reach resources on the home LAN that aren't running Tailscale
- **Service exposure:** Any Service annotated with `tailscale.com/expose: "true"` gets a Tailscale hostname and becomes accessible over the mesh — this is how `ts-pihole-dns` works
- **Auth:** Configured via a Kubernetes Secret (`operator-oauth` in the `tailscale` namespace)

### Sync Intervals

| Resource | Interval |
|---|---|
| GitRepository (Git poll) | 1 minute |
| Root Kustomization | 10 minutes |
| Pi-hole Kustomization | 1 minute |
| Pi-hole HelmRelease | 1 hour |
| Tailscale Kustomization | 10 minutes |
| Tailscale HelmRelease | 1 hour |

## Adding a New App

Here's the general pattern:

1. Create a namespace under `apps/<app-name>/namespace.yaml`
2. Add a HelmRepository (or plain YAML) — `apps/<app-name>/repository.yaml`
3. Create the HelmRelease — `apps/<app-name>/release.yaml`
4. Bundle them with a Kustomization — `apps/<app-name>/kustomization.yaml`
5. Wire it into the cluster by adding a Flux Kustomization — `clusters/homelab/app-<app-name>.yaml`

Flux picks it up automatically on the next reconciliation cycle.

## Roadmap

Things I'm planning to add:

- Cert-Manager + Let's Encrypt for TLS on the Pi-hole ingress
- A monitoring stack (Prometheus + Grafana)
- Media services (Plex, *arr suite)
- Home automation infrastructure
- Disaster recovery / backup automation

## License

MIT
