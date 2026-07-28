# Home Assistant Core Integration

## Overview

Deploy Home Assistant Core (single container, no Supervisor) into the homelab Kubernetes cluster, exposed via Traefik Ingress at `homeassistant.zieqs.online`. Pinned to the master node (`homelab`), with a 2Gi PVC for persistent configuration.

## Architecture

A single Deployment running `ghcr.io/home-assistant/home-assistant:stable` with a ClusterIP Service and standard Traefik Ingress. No companion services (MQTT, Zigbee2MQTT, ESPHome) — these can be added later as separate deployments.

## Files

```
apps/home-assistant/
  kustomization.yaml
  namespace.yaml
  home-assistant.yaml     # Deployment + Service + PVC + Ingress

clusters/homelab/
  app-home-assistant.yaml  # Flux Kustomization
```

## Manifest Details

### Namespace
- Name: `home-assistant`

### Deployment
- Image: `ghcr.io/home-assistant/home-assistant:stable`
- Node: `kubernetes.io/hostname: homelab` (master pinning)
- Container port: 8123
- Probes:
  - Liveness & Readiness: `HTTP GET /api/health` on port 8123
- Security:
  - `runAsUser: 1000`, `runAsGroup: 1000`, `runAsNonRoot: true`
  - `automountServiceAccountToken: false`
- Resources:
  - Requests: 256m CPU, 512Mi memory
  - Limits: 1 CPU, 1Gi memory
- `Recreate` strategy (stateful app, no rolling update needed)

### PVC
- Size: 2Gi
- Access mode: `ReadWriteOnce`
- Storage class: `local-path`

### Service
- Type: ClusterIP
- Port: 8123

### Ingress
- `ingressClassName: traefik`
- Host: `homeassistant.zieqs.online`
- No TLS config (Cloudflare edge terminates TLS upstream)

### Flux Kustomization
- Path: `./apps/home-assistant/`
- Interval: 5m
- Prune: true
- No SOPS decryption (no secrets)

## Out of Scope
- MQTT broker, Zigbee2MQTT, ESPHome (future additions)
- HA Supervisor/add-on system
- In-cluster TLS (handled by Cloudflare)
- PVC backups (HA has built-in backup to disk; external backup is future)
