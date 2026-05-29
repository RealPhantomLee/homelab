# homelab

[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D)](https://argo-cd.readthedocs.io/)
[![k3s](https://img.shields.io/badge/Kubernetes-k3s-FFC61C)](https://k3s.io/)
[![Tailscale](https://img.shields.io/badge/Overlay-Tailscale-242424)](https://tailscale.com/)
[![License](https://img.shields.io/badge/license-private-lightgrey)](#)

A four-node k3s cluster running on a Tailscale overlay network. Hosts self-hosted AI tooling, an encrypted note-sync server, a Kubernetes dashboard, and a public-facing static site — all managed via ArgoCD app-of-apps GitOps.

## Cluster Architecture

```
                          tailnet (100.x.x.x)
                          /        |        \
        +----+----+   +----+----+   +----+----+   +----+----+
        |          |   |          |   |          |   |          |
   +---------+  +---------+  +---------+  +---------+
   |blacknode|  |  aipi   |  |  core   |  |cyberdeck|
   | x86_64  |  | Pi 5 arm|  | x86_64  |  | Pi arm  |
   | control |  | worker  |  | worker  |  | worker  |
   | plane   |  |         |  |         |  | +Funnel |
   +---------+  +---------+  +---------+  +---------+
        |            |            |            |
        |  flannel-iface=tailscale0 — all pod traffic on tailnet
        |
   +---------+   +---------+   +---------+   +---------+
   | ArgoCD  |   |vault    |   |headlamp |   |phantom  |
   |local-ai |   |keeper   |   |dashboard|   |  site   |
   +---------+   +---------+   +---------+   +---------+
```

| Node      | Role          | Arch    | Tailscale IP   |
|-----------|---------------|---------|----------------|
| blacknode | control-plane | x86_64  | 100.90.234.21  |
| aipi      | worker (Pi 5) | arm64   | 100.76.179.112 |
| core      | worker        | x86_64  | 100.93.114.9   |
| cyberdeck | worker (Pi)   | arm64   | 100.121.96.1   |

Nodes communicate exclusively over Tailscale (`flannel-iface=tailscale0`). There is no LAN dependency — the cluster works wherever the nodes have tailnet connectivity.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full network and topology breakdown.

## Quick Start

Prerequisites: a fresh tailnet, three nodes joined to it, and access to your Tailscale OAuth credentials. Detailed walkthrough in [SETUP.md](SETUP.md).

```bash
# 1. Install k3s on blacknode (control plane), then join aipi + cyberdeck as workers.
# 2. Deploy the Tailscale Operator:
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
helm install tailscale-operator tailscale/tailscale-operator \
  --namespace tailscale --create-namespace \
  -f bootstrap/tailscale-operator/values.yaml

# 3. Install ArgoCD, then bootstrap the app-of-apps:
kubectl apply -n argocd -f clusters/homelab/apps/root-app.yaml

# 4. ArgoCD picks up clusters/homelab/apps/ and syncs every workload automatically.
```

## Repository Structure

```
apps/                       Kubernetes manifests per workload (Kustomize)
  phantom-site/             Static site served via Tailscale Funnel (cyberdeck)
  vaultkeeper/              Rust/Axum encrypted sync server (aipi)
  local-ai/                 AnythingLLM + Homarr + Ollama Endpoints (blacknode)
  headlamp/                 Kubernetes dashboard with cluster-admin access (core)
  monitoring/               Placeholder for Prometheus/Grafana
clusters/homelab/apps/      ArgoCD Application definitions (root-app + children)
bootstrap/                  One-time cluster setup (Tailscale Operator Helm values)
.github/workflows/          CI workflows
ARCHITECTURE.md             Network, nodes, GitOps, storage, exposure
SETUP.md                    Step-by-step initial bring-up
CONTRIBUTING.md             How to add a new workload
```

## Workloads

| Workload     | Namespace    | Pinned to | Exposure                              | Status      |
|--------------|--------------|-----------|---------------------------------------|-------------|
| phantom-site | phantom-site | cyberdeck | Tailscale Funnel (public HTTPS)       | Active      |
| vaultkeeper  | vaultkeeper  | aipi      | ClusterIP (tailnet exposure optional) | Disabled\*  |
| local-ai     | local-ai     | blacknode | Tailscale LoadBalancer (tailnet only) | Partial\*\* |
| headlamp     | kube-system  | core      | NodePort 30080 (tailnet only)         | Active      |
| monitoring   | monitoring   | (any)     | None yet                              | Placeholder |

\* `vaultkeeper-server` has `replicas: 0` pending a multi-arch image rebuild for arm64.
\*\* `anythingllm` and `homarr` have `replicas: 0` pending image publication; the `ollama-host` Endpoints object routes cluster traffic to native Ollama on blacknode (`100.90.234.21:11434`).

## Secrets

Nothing sensitive is committed to this repo. All Secrets (Homarr encryption key, GHCR pull credentials, Tailscale OAuth) are created imperatively with `kubectl create secret`. See [SETUP.md](SETUP.md#secrets).

## Documentation

- [ARCHITECTURE.md](ARCHITECTURE.md) — network topology, node roles, GitOps flow, storage, exposure patterns
- [SETUP.md](SETUP.md) — prerequisites, OAuth, k3s install, ArgoCD bootstrap, troubleshooting
- [CONTRIBUTING.md](CONTRIBUTING.md) — adding a new workload, Kustomize patterns, local testing
