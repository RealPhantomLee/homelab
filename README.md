# homelab

k3s cluster on Tailscale — self-hosted AI, encrypted note sync, and personal site.

## Cluster

| Node | Role | Arch | Tailscale IP |
|---|---|---|---|
| blacknode | control-plane | x86_64 | 100.90.234.21 |
| aipi | worker (Pi 5) | arm64 | 100.76.179.112 |
| cyberdeck | worker (Pi, Funnel) | arm64 | 100.121.96.1 |

Nodes communicate exclusively over Tailscale (`flannel-iface=tailscale0`). No LAN dependency.

## Stack

| Layer | Tool |
|---|---|
| Kubernetes | k3s |
| Overlay network | Tailscale |
| Service exposure | Tailscale Operator (LoadBalancer) + Funnel |
| GitOps | ArgoCD (app-of-apps) |
| Manifests | Kustomize |

## Workloads

- **local-ai** — AnythingLLM + Homarr on blacknode, Ollama runs native on host
- **vaultkeeper** — Rust/Axum encrypted sync server on aipi (Pi 5)
- **phantom-site** — phantomcybersolutions.com served via Tailscale Funnel on cyberdeck

## Repo Layout

```
apps/              # Kubernetes manifests per workload
clusters/homelab/  # ArgoCD Application definitions
bootstrap/         # One-time cluster setup (Tailscale Operator Helm values)
```

## Bootstrap Order

See [bootstrap/README.md](bootstrap/README.md) for the full step-by-step.

1. k3s on blacknode (control plane)
2. k3s on aipi + cyberdeck (workers, via Tailscale IPs)
3. Tailscale Operator (Helm)
4. ArgoCD + apply `clusters/homelab/apps/root-app.yaml`
5. ArgoCD syncs all workloads automatically

## Secrets

All secrets are created imperatively with `kubectl create secret` — never committed to this repo.
