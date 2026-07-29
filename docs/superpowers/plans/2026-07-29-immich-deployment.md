# Immich Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Immich (self-hosted photo/video management) to the homelab k3s cluster using the official Helm chart, managed via Flux CD.

**Architecture:** Official Immich Helm chart with bundled PostgreSQL and Redis subcharts. Media library stored on master node hostPath (`/mnt/HDD2/immich`). Traefik ingress at `immich.zieqs.online`. SOPS-encrypted secrets for DB passwords.

**Tech Stack:** Kubernetes, Flux CD (kustomize-controller + helm-controller), Immich Helm chart, Traefik, SOPS/age

## Global Constraints

- All new files go in `apps/immich/` directory (Kustomize overlay pattern)
- Flux Kustomization goes in `clusters/homelab/app-immich.yaml`
- HostPath PV uses node affinity for `homelab` master node
- Secrets use SOPS encryption with age (key in `age.agekey`)
- Ingress uses `traefik` ingressClassName, HTTP only (Cloudflare TLS upstream)
- All services pinned to master node via `nodeSelector`
- ML service enabled (CPU mode)
- Follow existing pihole/vaultwarden Helm patterns exactly

---

## File Map

| File | Purpose |
|------|---------|
| `apps/immich/namespace.yaml` | `immich` namespace |
| `apps/immich/repository.yaml` | HelmRepository pointing at immich-charts |
| `apps/immich/release.yaml` | HelmRelease with all values config |
| `apps/immich/storage.yaml` | hostPath PV + PVC for media library |
| `apps/immich/secrets/immich-secret.yaml` | Plaintext secret (gitignored) with DB passwords |
| `apps/immich/secrets/immich-secret.enc.yaml` | SOPS-encrypted version (committed) |
| `apps/immich/kustomization.yaml` | Kustomize listing all resources |
| `clusters/homelab/app-immich.yaml` | Flux Kustomization pointing at `./apps/immich` |

---

### Task 1: Create namespace, repository, and kustomization skeleton

**Files:**
- Create: `apps/immich/namespace.yaml`
- Create: `apps/immich/repository.yaml`
- Create: `apps/immich/kustomization.yaml`

**Interfaces:**
- Consumes: nothing
- Produces: namespace `immich`, HelmRepository `immich-charts`, kustomization scaffold

- [ ] **Step 1: Create `apps/immich/` directory**

```bash
mkdir -p apps/immich/secrets
```

- [ ] **Step 2: Create `apps/immich/namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: immich
```

- [ ] **Step 3: Create `apps/immich/repository.yaml`**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: immich-charts
  namespace: flux-system
spec:
  interval: 1h
  url: https://immich-app.github.io/immich-charts
```

- [ ] **Step 4: Create `apps/immich/kustomization.yaml`** (initially with namespace only, will add resources as tasks complete)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace.yaml
- repository.yaml
```

- [ ] **Step 5: Validate kustomize builds**

```bash
kustomize build apps/immich
```
Expected: namespace + HelmRepository rendered successfully.

---

### Task 2: Create storage resources (hostPath PV + PVC)

**Files:**
- Create: `apps/immich/storage.yaml`

**Interfaces:**
- Consumes: namespace `immich` (Task 1)
- Produces: PV `immich-library-pv` (hostPath at `/mnt/HDD2/immich`), PVC `immich-library-pvc`

- [ ] **Step 1: Create `apps/immich/storage.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: immich-library-pv
spec:
  capacity:
    storage: 5Ti
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/HDD2/immich
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - homelab
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: immich-library-pvc
  namespace: immich
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Ti
```

- [ ] **Step 2: Add `storage.yaml` to `kustomization.yaml`**

```yaml
resources:
- namespace.yaml
- repository.yaml
- storage.yaml
```

- [ ] **Step 3: Validate kustomize builds**

```bash
kustomize build apps/immich
```
Expected: namespace + HelmRepository + PV + PVC rendered successfully.

---

### Task 3: Create Helm release with all values

**Files:**
- Create: `apps/immich/release.yaml`

**Interfaces:**
- Consumes: HelmRepository `immich-charts` (Task 1), PVC `immich-library-pvc` (Task 2), Secret `immich-secret` (Task 4)
- Produces: HelmRelease `immich`

- [ ] **Step 1: Create `apps/immich/release.yaml`**

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: immich
  namespace: immich
spec:
  releaseName: immich
  chart:
    spec:
      chart: immich
      version: ">= 0.1.0"
      sourceRef:
        kind: HelmRepository
        name: immich-charts
        namespace: flux-system
  interval: 10m
  postRenderers:
    - kustomize:
        patches:
          - target:
              kind: Deployment
            patch: |
              - op: add
                path: /spec/template/spec/automountServiceAccountToken
                value: false
  values:
    image:
      tag: "v1.120.2"

    immich:
      metrics:
        enabled: false

    persistence:
      library:
        existingClaim: immich-library-pvc

    postgresql:
      enabled: true
      global:
        postgresql:
          auth:
            database: immich
            username: immich
      primary:
        persistence:
          storageClass: "local-path"
          size: 10Gi

    redis:
      enabled: true
      auth:
        enabled: true
      master:
        persistence:
          storageClass: "local-path"
          size: 1Gi

    server:
      resources:
        requests:
          cpu: 100m
          memory: 512Mi

    microservices:
      resources:
        requests:
          cpu: 100m
          memory: 512Mi

    machine-learning:
      enabled: true
      resources:
        requests:
          cpu: 100m
          memory: 512Mi

    ingress:
      main:
        enabled: true
        ingressClassName: traefik
        annotations:
          traefik.ingress.kubernetes.io/router.entrypoints: web
        hosts:
          - host: immich.zieqs.online
            paths:
              - path: /
                pathType: Prefix

    nodeSelector:
      kubernetes.io/hostname: homelab
  valuesFrom:
    - kind: Secret
      name: immich-secret
      valuesKey: postgres-password
      targetPath: postgresql.global.postgresql.auth.password
    - kind: Secret
      name: immich-secret
      valuesKey: redis-password
      targetPath: redis.auth.password
```

- [ ] **Step 2: Add `release.yaml` to `kustomization.yaml`**

```yaml
resources:
- namespace.yaml
- repository.yaml
- storage.yaml
- release.yaml
```

- [ ] **Step 3: Validate kustomize builds**

```bash
kustomize build apps/immich
```
Expected: namespace + HelmRepository + PV + PVC + HelmRelease rendered.

---

### Task 4: Create SOPS-encrypted secrets

**Files:**
- Create: `apps/immich/secrets/immich-secret.yaml` (plaintext, gitignored)
- Create: `apps/immich/secrets/immich-secret.enc.yaml` (SOPS encrypted, committed)

**Interfaces:**
- Consumes: nothing
- Produces: Secret `immich-secret` (in namespace `immich`) consumed by Task 3 HelmRelease

- [ ] **Step 1: Generate random passwords**

```bash
POSTGRES_PASSWORD=$(openssl rand -base64 24)
REDIS_PASSWORD=$(openssl rand -base64 24)
echo "Postgres password: $POSTGRES_PASSWORD"
echo "Redis password: $REDIS_PASSWORD"
```

- [ ] **Step 2: Create `apps/immich/secrets/immich-secret.yaml`**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: immich-secret
  namespace: immich
type: Opaque
stringData:
  postgres-password: "${POSTGRES_PASSWORD}"
  redis-password: "${REDIS_PASSWORD}"
```

(Replace `${POSTGRES_PASSWORD}` and `${REDIS_PASSWORD}` with actual generated values)

- [ ] **Step 3: Encrypt the secret with SOPS**

```bash
sops --encrypt apps/immich/secrets/immich-secret.yaml > apps/immich/secrets/immich-secret.enc.yaml
```

- [ ] **Step 4: Verify the encrypted file is valid YAML**

```bash
python3 -c "import yaml; yaml.safe_load(open('apps/immich/secrets/immich-secret.enc.yaml')); print('valid')"
```

- [ ] **Step 5: Add secret to `kustomization.yaml`**

```yaml
resources:
- namespace.yaml
- repository.yaml
- storage.yaml
- release.yaml
- secrets/immich-secret.enc.yaml
```

- [ ] **Step 6: Validate kustomize builds**

```bash
kustomize build apps/immich
```
Expected: All 5 resources rendered successfully.

---

### Task 5: Create Flux Kustomization

**Files:**
- Create: `clusters/homelab/app-immich.yaml`

**Interfaces:**
- Consumes: Kustomize overlay at `./apps/immich` (Tasks 1-4)
- Produces: Flux Kustomization reconciling Immich

- [ ] **Step 1: Create `clusters/homelab/app-immich.yaml`**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: immich-root
  namespace: flux-system
spec:
  interval: 5m0s
  path: ./apps/immich
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

- [ ] **Step 2: Validate full cluster overlay builds**

```bash
kustomize build clusters/homelab
```
Expected: Success — the new immich Kustomization is included.

---

### Task 6: Final validation

- [ ] **Step 1: Run kustomize build on apps/immich**

```bash
kustomize build apps/immich
```

Expected: Clean output with all 5 resources (namespace, repository, storage, release, secret).

- [ ] **Step 2: Run GitHub Actions validation workflow check**

Ensure the `kustomize build` output passes `kubeconform` validation (mimicking CI).

- [ ] **Step 3: Ensure all files are tracked correctly**

```bash
git status
```

Verify that `immich-secret.yaml` (plaintext) is NOT tracked (should be in `.gitignore`), and `immich-secret.enc.yaml` IS tracked.

---

### Task 7: Commit and push

- [ ] **Step 1: Commit all changes**

```bash
git add apps/immich/ clusters/homelab/app-immich.yaml docs/superpowers/plans/2026-07-29-immich-deployment.md docs/superpowers/specs/2026-07-29-immich-design.md
git commit -m "feat: add Immich deployment with Helm chart"
```

- [ ] **Step 2: Verify commit is clean**

```bash
git log --oneline -3
git status
```
