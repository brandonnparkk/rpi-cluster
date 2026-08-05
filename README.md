# RPI Homelab

A GitOps-managed homelab Kubernetes cluster (Raspberry Pi) that uses FluxCD to sync deployments from this repo, SOPS/age for secret encryption, and Cloudflare Tunnel for ingress. Currently runs a self-hosted linkding bookmark manager and a kube-prometheus-stack observability stack.
![homepage](./linkding.png)


## Architecture

![rpi-cluster architecture](./architecture.svg)

## Tech Stack

- **GitOps**: FluxCD (GitRepository + Kustomization controllers)
- **Compute**: Raspberry Pi Kubernetes cluster
- **Config management**: Kustomize (base/overlay pattern)
- **Package management**: Helm (via Flux `HelmRelease`/`HelmRepository`)
- **Secrets**: SOPS encryption with age
- **Ingress**: Cloudflare Tunnel (`cloudflared`)
- **Monitoring**: kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
- **Apps**: linkding (self-hosted bookmark manager)

## Prerequisites

- A Kubernetes cluster with `kubectl` access
- [Flux CLI](https://fluxcd.io/flux/installation/) bootstrapped against this repo
- [SOPS](https://github.com/getsops/sops) and [age](https://github.com/FiloSottile/age) installed, with the cluster's age private key available for decryption
- A Cloudflare account and tunnel credentials for ingress
- SSH access configured for `git@github.com` (Flux syncs over SSH)

## Directory Structure
[FluxCD's Monorepo Structure](https://fluxcd.io/flux/guides/repository-structure/)
```
rpi-cluster/
├── clusters/
│   └── staging/
│       ├── flux-system/         # Flux-managed bootstrap manifests (gotk-components, gotk-sync)
│       ├── apps.yaml             # Kustomization: syncs apps/staging, SOPS decryption enabled
│       ├── monitoring.yaml       # Kustomization: syncs monitoring/controllers/staging
│       └── .sops.yaml            # SOPS encryption rules (age recipient, encrypted_regex)
├── apps/
│   ├── base/
│   │   └── linkding/             # namespace, deployment, service, PVC
│   └── staging/
│       └── linkding/             # base overlay + cloudflared tunnel + encrypted secrets
├── monitoring/
│   └── controllers/
│       ├── base/
│       │   └── kube-prometheus-stack/   # namespace, HelmRepository, HelmRelease
│       └── staging/
│           └── kube-prometheus-stack/   # overlay referencing base
└── README.md
```

## Design Decisions

| Decision | Chose | Over | Why |
|---|---|---|---|
| Compute | Raspberry Pi 5 (8GB) | EC2, managed Kubernetes | Own the hardware, no recurring cost, forces hands-on control plane work |
| GitOps engine | Flux CD | Argo CD | Smaller footprint on a constrained node, native SOPS decryption in kustomize-controller |
| Own manifests | Kustomize overlays | Helm charts | No templating indirection over YAML I already control |
| Third-party software | Helm via HelmRelease | Vendored or rendered manifests | Versioned and pinnable, no re-vendoring on every upstream release |
| Helm controller | Flux helm-controller | k3s bundled helm-controller | Conflicting `HelmChart` CRDs; disabled k3s's via `--disable=helm-controller` |
| Secrets | SOPS + age | Sealed Secrets, External Secrets Operator | No external store to run, decryptable locally, age avoids GPG keyring management |
| External access | Cloudflare Tunnel | Port forwarding, Ingress + cert-manager | Outbound-only, no inbound firewall rules, no static IP, TLS at the edge |
| Service exposure | ClusterIP | LoadBalancer, NodePort | cloudflared runs in-cluster, so nothing needs LAN or WAN reachability |
| Monitoring | kube-prometheus-stack | Uptime Kuma, Beszel, Grafana Cloud | ServiceMonitor CRDs make scrape config declarative and GitOps-native |
| Control plane scraping | Disabled controller-manager, scheduler, proxy, etcd monitors | Chart defaults | k3s consolidates these into one process and uses SQLite, so defaults yield dead targets and false alerts |
| Node networking | Wired ethernet | WiFi | WiFi power management causes the node to flap NotReady |

For a full writeup on trade-offs and design rationale, see the [blog post](https://bpark.dev/blog/2026-07-31-rpi-homelab/).