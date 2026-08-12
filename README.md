# RPI Homelab

A GitOps-managed homelab Kubernetes cluster (Raspberry Pi) that uses FluxCD to sync deployments from this repo, SOPS/age for secret encryption, and Traefik Ingress for routing. Currently runs a self-hosted linkding bookmark manager, a kube-prometheus-stack observability stack, and Renovate for automated dependency updates.
![homepage](./linkding.png)


## Architecture

![rpi-cluster architecture](./architecture.svg)

> Note: this diagram reflects an earlier state (Cloudflare Tunnel as the ingress path, no Renovate/infrastructure). It hasn't been regenerated for the current Traefik-based setup yet.

## Tech Stack

- **GitOps**: FluxCD (GitRepository + Kustomization controllers)
- **Compute**: Raspberry Pi Kubernetes cluster (k3s)
- **Config management**: Kustomize (base/overlay pattern)
- **Package management**: Helm (via Flux `HelmRelease`/`HelmRepository`)
- **Secrets**: SOPS encryption with age
- **Ingress**: Traefik (bundled with k3s), via native `Ingress` resources
- **External access**: Cloudflare Tunnel (`cloudflared`), currently disabled in favor of local Traefik ingress
- **Monitoring**: kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
- **Dependency automation**: Renovate, scheduled via `CronJob`, opens PRs against this repo
- **Apps**: linkding (self-hosted bookmark manager)

## Prerequisites

- A Kubernetes cluster with `kubectl` access (k3s, with the bundled Traefik ingress controller)
- [Flux CLI](https://fluxcd.io/flux/installation/) bootstrapped against this repo
- [SOPS](https://github.com/getsops/sops) and [age](https://github.com/FiloSottile/age) installed, with the cluster's age private key available for decryption
- A GitHub token for Renovate to open PRs against this repo
- (Optional) A Cloudflare account and tunnel credentials, only needed if `cloudflare.yaml` is re-enabled for public ingress
- SSH access configured for `git@github.com` (Flux syncs over SSH)

## Directory Structure
[FluxCD's Monorepo Structure](https://fluxcd.io/flux/guides/repository-structure/)
```
rpi-cluster/
├── clusters/
│   └── staging/
│       ├── flux-system/         # Flux-managed bootstrap manifests (gotk-components, gotk-sync)
│       ├── apps.yaml             # Kustomization: syncs apps/staging, SOPS decryption enabled
│       ├── monitoring.yaml       # Kustomizations: monitoring-controllers + monitoring-configs (SOPS enabled)
│       ├── infrastructure.yaml   # Kustomization: syncs infrastructure/controllers/staging, SOPS decryption enabled
│       └── .sops.yaml            # SOPS encryption rules (age recipient, encrypted_regex)
├── apps/
│   ├── base/
│   │   └── linkding/             # namespace, deployment, service, PVC
│   └── staging/
│       └── linkding/             # base overlay + Traefik ingress + encrypted secrets (cloudflare.yaml disabled)
├── monitoring/
│   ├── controllers/
│   │   ├── base/
│   │   │   └── kube-prometheus-stack/   # namespace, HelmRepository, HelmRelease (Grafana ingress + TLS values)
│   │   └── staging/
│   │       └── kube-prometheus-stack/   # overlay referencing base
│   └── configs/
│       └── staging/
│           └── kube-prometheus-stack/   # encrypted Grafana TLS secret
├── infrastructure/
│   └── controllers/
│       ├── base/
│       │   └── renovate/         # namespace, configmap, cronjob, encrypted env secret
│       └── staging/
│           └── renovate/         # overlay referencing base, namespace: renovate
├── renovate.json                 # Renovate bot configuration
├── architecture.svg
└── README.md
```

## Design Decisions

| Decision | Chose | Over | Why |
|---|---|---|---|
| Compute | Raspberry Pi 5 (8GB) | EC2, managed Kubernetes | Own the hardware, no recurring cost, forces hands-on control plane work |
| GitOps engine | Flux CD | Argo CD | Smaller footprint on a constrained node, native SOPS decryption in kustomize-controller |
| Helm controller | Flux helm-controller | k3s bundled helm-controller | Conflicting `HelmChart` CRDs; disabled k3s's via `--disable=helm-controller` |
| Secrets | SOPS + age | Sealed Secrets, External Secrets Operator | No external store to run, decryptable locally, age avoids GPG keyring management |
| External access | Cloudflare Tunnel | Port forwarding, Ingress + cert-manager | Outbound-only, no inbound firewall rules, no static IP, TLS at the edge |
| Service exposure | ClusterIP | LoadBalancer, NodePort | cloudflared runs in-cluster, so nothing needs LAN or WAN reachability |
| Monitoring | kube-prometheus-stack | Uptime Kuma, Beszel, Grafana Cloud | ServiceMonitor CRDs make scrape config declarative and GitOps-native |

For a full writeup on trade-offs and design rationale, see the [blog post](https://bpark.dev/blog/2026-07-31-rpi-homelab/).