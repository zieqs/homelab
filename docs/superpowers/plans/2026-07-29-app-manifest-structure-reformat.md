# App Manifest Structure Reformat — Implementation Plan

> **For agentic workers:** Use inline execution. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Standardize all 12 app/infra directories to use `deployment.yaml`, `networking.yaml`, `storage.yaml`, `kustomization.yaml`, and (for Helm charts) `repository.yaml` + `release.yaml`.

**Architecture:** No functional changes — only file renames and resource-by-kind splitting/merging. Each app directory is processed independently.

**Tech Stack:** Kubernetes YAML manifests, Kustomize, Flux, SOPS.

## Global Constraints

- No functional changes to resource content — only file organization
- `namespace.yaml` stays separate in every app
- `secrets/` subdirectory pattern stays unchanged
- `clusters/homelab/` Flux Kustomizations unchanged (they reference app paths)
- Helm apps: `repository.yaml` + `release.yaml` naming
- After each task: verify with `git diff --stat` then `kustomize build clusters/homelab/`

---

### Task 1: glances + samba + minecraft

**Files:**
- Modify: `apps/glances/`, `apps/samba/`, `apps/minecraft/`

**Description:** Three smallest apps, each splits one YAML file into deployment.yaml + networking.yaml. No secrets, no storage, no ingress in samba/minecraft.

- [ ] **Step 1: Convert glances**
  - Read `apps/glances/glances.yaml` — extract Deployments into `apps/glances/deployment.yaml`, Services into `apps/glances/networking.yaml`
  - Read `apps/glances/ingress.yaml` — append Ingress(es) to `apps/glances/networking.yaml`
  - Update `apps/glances/kustomization.yaml` — change `glances.yaml` → `deployment.yaml`, keep `ingress.yaml` but change to `networking.yaml`
  - Remove `apps/glances/glances.yaml` and `apps/glances/ingress.yaml`

- [ ] **Step 2: Convert samba**
  - Read `apps/samba/samba.yaml` — extract Deployment into `apps/samba/deployment.yaml`, Service into `apps/samba/networking.yaml`
  - Update `apps/samba/kustomization.yaml` — change `samba.yaml` → `deployment.yaml`, add `networking.yaml`
  - Remove `apps/samba/samba.yaml`

- [ ] **Step 3: Convert minecraft**
  - Read `apps/minecraft/minecraft.yaml` — extract Deployment into `apps/minecraft/deployment.yaml`, NodePort Service into `apps/minecraft/networking.yaml`
  - Update `apps/minecraft/kustomization.yaml` — change `minecraft.yaml` → `deployment.yaml`, add `networking.yaml`
  - Remove `apps/minecraft/minecraft.yaml`

- [ ] **Step 4: Verify Task 1**
  - Run: `git diff --stat` — should show only renames/creates, no content changes
  - Run: `kustomize build clusters/homelab/ --enable-helm` to validate all paths
  - Run: `git add -A && git commit -m "refactor: standardize glances, samba, minecraft file structure"`

---

### Task 2: home-assistant

**Files:**
- Modify: `apps/home-assistant/`

**Description:** Splits home-assistant.yaml (PVC + Deployment + Service) into storage.yaml + deployment.yaml + networking.yaml. Merges ingress.yaml into networking.yaml. No secrets.

- [ ] **Step 1: Convert home-assistant**
  - Read `apps/home-assistant/home-assistant.yaml` — extract PVC into `apps/home-assistant/storage.yaml`, Deployment into `apps/home-assistant/deployment.yaml`, Service into `apps/home-assistant/networking.yaml`
  - Read `apps/home-assistant/ingress.yaml` — append Ingress to `apps/home-assistant/networking.yaml`
  - Update `apps/home-assistant/kustomization.yaml` — change `home-assistant.yaml` → `deployment.yaml`, `storage.yaml`, `networking.yaml`; remove `ingress.yaml`
  - Remove `apps/home-assistant/home-assistant.yaml` and `apps/home-assistant/ingress.yaml`

- [ ] **Step 2: Verify Task 2**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize home-assistant file structure"`

---

### Task 3: homarr

**Files:**
- Modify: `apps/homarr/`

**Description:** Splits homarr.yaml (PVC + Deployment + Service) into storage.yaml + deployment.yaml + networking.yaml. Merges ingress.yaml into networking.yaml. Has secrets subdirectory (untouched).

- [ ] **Step 1: Convert homarr**
  - Read `apps/homarr/homarr.yaml` — extract PVC into `apps/homarr/storage.yaml`, Deployment into `apps/homarr/deployment.yaml`, Service into `apps/homarr/networking.yaml`
  - Read `apps/homarr/ingress.yaml` — append Ingress to `apps/homarr/networking.yaml`
  - Update `apps/homarr/kustomization.yaml` — change `homarr.yaml` → `deployment.yaml`, `storage.yaml`, `networking.yaml`; remove `ingress.yaml`; keep secrets entry
  - Remove `apps/homarr/homarr.yaml` and `apps/homarr/ingress.yaml`

- [ ] **Step 2: Verify Task 3**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize homarr file structure"`

---

### Task 4: obsidian

**Files:**
- Modify: `apps/obsidian/`

**Description:** Splits couchdb.yaml (PVC + Deployment + Service) into storage.yaml + deployment.yaml + networking.yaml. Merges ingress.yaml into networking.yaml. Has secrets subdirectory (untouched).

- [ ] **Step 1: Convert obsidian**
  - Read `apps/obsidian/couchdb.yaml` — extract PVC into `apps/obsidian/storage.yaml`, Deployment into `apps/obsidian/deployment.yaml`, Service into `apps/obsidian/networking.yaml`
  - Read `apps/obsidian/ingress.yaml` — append Ingress to `apps/obsidian/networking.yaml`
  - Update `apps/obsidian/kustomization.yaml` — change `couchdb.yaml` → `deployment.yaml`, `storage.yaml`, `networking.yaml`; remove `ingress.yaml`; keep secrets entry
  - Remove `apps/obsidian/couchdb.yaml` and `apps/obsidian/ingress.yaml`

- [ ] **Step 2: Verify Task 4**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize obsidian file structure"`

---

### Task 5: media

**Files:**
- Modify: `apps/media/`

**Description:** Merges three files (jellyfin.yaml, arr-stack.yaml, torrent-stack.yaml) into deployment.yaml (all Deployments) + networking.yaml (all Services + Ingresses). No storage.

- [ ] **Step 1: Convert media**
  - Read `apps/media/jellyfin.yaml` — extract Deployment → deployment.yaml, Service+Ingress → networking.yaml
  - Read `apps/media/arr-stack.yaml` — extract Deployments (Radarr, Sonarr, Jellyseerr, Bazarr) → deployment.yaml, Services+Ingress → networking.yaml
  - Read `apps/media/torrent-stack.yaml` — extract Deployments (qBittorrent, Prowlarr) → deployment.yaml, Services+Ingress → networking.yaml
  - Update `apps/media/kustomization.yaml` — change `jellyfin.yaml`, `arr-stack.yaml`, `torrent-stack.yaml` → `deployment.yaml`, `networking.yaml`
  - Remove `apps/media/jellyfin.yaml`, `apps/media/arr-stack.yaml`, `apps/media/torrent-stack.yaml`

- [ ] **Step 2: Verify Task 5**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize media file structure"`

---

### Task 6: pihole (Helm)

**Files:**
- Modify: `apps/pihole/`

**Description:** Helm app. Renames ingress.yaml → networking.yaml. release.yaml and repository.yaml stay. Has secrets subdirectory (untouched).

- [ ] **Step 1: Convert pihole**
  - Read `apps/pihole/ingress.yaml` — write as `apps/pihole/networking.yaml`
  - Update `apps/pihole/kustomization.yaml` — change `ingress.yaml` → `networking.yaml`; keep `release.yaml`, `repository.yaml`, secrets
  - Remove `apps/pihole/ingress.yaml`

- [ ] **Step 2: Verify Task 6**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize pihole file structure"`

---

### Task 7: vaultwarden (Helm)

**Files:**
- Modify: `apps/vaultwarden/`

**Description:** Helm app. vaultwarden.yaml contains both PVC and HelmRelease — extract PVC to storage.yaml, HelmRelease to release.yaml. repository.yaml stays.

- [ ] **Step 1: Convert vaultwarden**
  - Read `apps/vaultwarden/vaultwarden.yaml` — extract PVC into `apps/vaultwarden/storage.yaml`, HelmRelease into `apps/vaultwarden/release.yaml`
  - Update `apps/vaultwarden/kustomization.yaml` — change `vaultwarden.yaml` → `storage.yaml`, `release.yaml`; keep `repository.yaml`
  - Remove `apps/vaultwarden/vaultwarden.yaml`

- [ ] **Step 2: Verify Task 7**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize vaultwarden file structure"`

---

### Task 8: infrastructure/cloudflared

**Files:**
- Modify: `infrastructure/cloudflared/`

**Description:** Infrastructure app with secrets. Renames cloudflared.yaml → deployment.yaml.

- [ ] **Step 1: Convert cloudflared**
  - Read `infrastructure/cloudflared/cloudflared.yaml` — write as `infrastructure/cloudflared/deployment.yaml`
  - Update `infrastructure/cloudflared/kustomization.yaml` — change `cloudflared.yaml` → `deployment.yaml`; keep secrets
  - Remove `infrastructure/cloudflared/cloudflared.yaml`

- [ ] **Step 2: Verify Task 8**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize cloudflared file structure"`

---

### Task 9: infrastructure/radar + tailscale (Helm)

**Files:**
- Modify: `infrastructure/radar/`, `infrastructure/tailscale/`

**Description:** Two infra Helm apps. radar: rename radar.yaml → release.yaml. tailscale: pihole-tailscale.yaml → networking.yaml. Both keep repository.yaml.

- [ ] **Step 1: Convert radar**
  - Read `infrastructure/radar/radar.yaml` — rename to `infrastructure/radar/release.yaml`
  - Update `infrastructure/radar/kustomization.yaml` — change `radar.yaml` → `release.yaml`; keep `repository.yaml`
  - Remove `infrastructure/radar/radar.yaml`

- [ ] **Step 2: Convert tailscale**
  - Read `infrastructure/tailscale/pihole-tailscale.yaml` — write as `infrastructure/tailscale/networking.yaml`
  - Update `infrastructure/tailscale/kustomization.yaml` — change `pihole-tailscale.yaml` → `networking.yaml`; keep `release.yaml`, `repository.yaml`
  - Remove `infrastructure/tailscale/pihole-tailscale.yaml`

- [ ] **Step 3: Verify Task 9**
  - Run: `git diff --stat`
  - Run: `kustomize build clusters/homelab/ --enable-helm`
  - Run: `git add -A && git commit -m "refactor: standardize radar and tailscale file structure"`

---

### Task 10: Write spec doc and commit

**Files:**
- Create: `docs/superpowers/specs/2026-07-29-app-manifest-structure-reformat-design.md`

- [ ] **Step 1: Write the spec document** (already done above)

- [ ] **Step 2: Commit**

```bash
git add docs/superpowers/specs/2026-07-29-app-manifest-structure-reformat-design.md
git commit -m "docs: add reformat spec"
```

- [ ] **Step 3: Save the plan document (**if not already committed with last task):
  This step is implicit; the plan will be committed with the last task.
