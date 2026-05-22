# Architecture

This document covers the network topology, node roles, GitOps flow, storage strategy, and service exposure patterns for the homelab cluster.

## Network Topology

The cluster runs on a Tailscale overlay — every node-to-node and pod-to-pod packet flows across the tailnet rather than a physical LAN.

```
                            Tailscale tailnet
                          (100.64.0.0/10 CGNAT)
                                  |
                                  |  WireGuard, end-to-end encrypted
                                  |
   100.90.234.21          100.76.179.112             100.121.96.1
   +-----------+          +-------------+            +------------+
   | blacknode |<-------->|    aipi     |<---------->| cyberdeck  |
   |  x86_64   |          |   Pi 5 arm  |            |  Pi arm    |
   | control   |          |   worker    |            |  worker    |
   | plane     |          |             |            | + Funnel   |
   +-----------+          +-------------+            +------------+
        ^                       ^                         ^
        |                       |                         |
        +-----------------------+-------------------------+
                          flannel-iface=tailscale0
                          (pod CIDR over WireGuard)
```

k3s on each node is configured with `--flannel-iface=tailscale0`, forcing the in-cluster pod network to ride the tailnet interface. Nodes do not need to share a local subnet — the cluster survives anywhere the nodes can reach the tailnet.

## Node Roles

### blacknode (control-plane, x86_64)

- Tailscale IP: `100.90.234.21`
- Architecture: `linux/amd64`
- Role: k3s server (control plane + worker)
- Hosts: ArgoCD, Tailscale Operator, `local-ai` namespace (AnythingLLM, Homarr)
- Native services: Ollama listens on the host at `100.90.234.21:11434` (not in-cluster). The `ollama-host` Service in `local-ai` uses a manual Endpoints object to route cluster traffic to it.

### aipi (worker, Pi 5 arm64)

- Tailscale IP: `100.76.179.112`
- Architecture: `linux/arm64`
- Role: k3s agent
- Hosts: `vaultkeeper-server` (pinned via `nodeSelector: kubernetes.io/hostname: aipi`)
- Storage: PersistentVolumeClaims for `vaultkeeper-data` (5Gi) and `vaultkeeper-backups` (10Gi) are local-path provisioned on this node's disk.

### cyberdeck (worker, Pi arm64, Funnel-enabled)

- Tailscale IP: `100.121.96.1`
- Architecture: `linux/arm64`
- Role: k3s agent with `funnel=enabled` node label
- Hosts: `phantom-site` (pinned via `nodeSelector: kubernetes.io/hostname: cyberdeck`)
- The only node permitted to terminate Tailscale Funnel (public internet) traffic.

## GitOps Flow

ArgoCD watches this repo and reconciles cluster state on every commit to `main`. The "app-of-apps" pattern means a single root Application owns every other Application.

```
  git push -> main
       |
       v
  GitHub: RealPhantomLee/homelab
       |
       v
  ArgoCD root Application (clusters/homelab/apps/root-app.yaml)
       |
       |  watches clusters/homelab/apps/ directory
       |
       +--> Application: phantom-site -> apps/phantom-site/
       +--> Application: vaultkeeper  -> apps/vaultkeeper/
       +--> Application: local-ai     -> apps/local-ai/
       +--> Application: monitoring   -> apps/monitoring/
                              |
                              v
                    Kustomize build -> kubectl apply
                              |
                              v
                       Workload pods on
                       blacknode / aipi / cyberdeck
```

All Applications use:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

That means: drift is corrected automatically, deleted resources are pruned, and namespaces are created on first sync. Cascading deletes are enabled via the `resources-finalizer.argocd.io` finalizer on each Application.

## Storage Strategy

The cluster uses k3s's built-in `local-path` StorageClass — no Longhorn, Ceph, or NFS. Each PVC binds to a hostPath directory on whichever node the pod schedules to.

| PVC                   | Size  | Bound to (via nodeSelector) | Purpose                    |
|-----------------------|-------|-----------------------------|----------------------------|
| `vaultkeeper-data`    | 5Gi   | aipi                        | SQLite, configs            |
| `vaultkeeper-backups` | 10Gi  | aipi                        | Snapshots, exports         |
| `homarr-appdata`      | 5Gi   | blacknode                   | Homarr configs             |
| `anythingllm-storage` | (TBD) | blacknode                   | AnythingLLM state          |

Trade-offs:

- **Pros**: zero operational overhead, fast (local NVMe/SD), works out of the box on k3s.
- **Cons**: no replication, no automatic failover. If a node dies, its PVCs are lost. Workloads are pinned via `nodeSelector` so PVCs always land on the intended node — moving a workload between nodes requires manual data migration.

Backups for stateful workloads are the application's responsibility (e.g. VaultKeeper writes to its own `backups` PVC).

## Service Exposure Patterns

Three exposure tiers, all driven by the Tailscale Operator:

### 1. ClusterIP (internal only)

Default. Used for `vaultkeeper-server` (currently), `ollama-host`, and any service that should not leak outside the cluster.

### 2. Tailscale LoadBalancer (tailnet-only)

Services with `spec.loadBalancerClass: tailscale` are picked up by the Tailscale Operator, which provisions a tailnet device per service. The annotation `tailscale.com/hostname` controls the MagicDNS name.

```yaml
spec:
  type: LoadBalancer
  loadBalancerClass: tailscale
metadata:
  annotations:
    tailscale.com/hostname: "anythingllm"  # reachable at anythingllm.<tailnet>.ts.net
```

Used by `anythingllm` and `homarr`. Only devices on the tailnet can reach these.

### 3. Tailscale Funnel (public internet)

Same as the LoadBalancer pattern, but with the additional annotation `tailscale.com/funnel: "true"`. Funnel terminates public HTTPS on Tailscale's edge and tunnels it to the service. Only services scheduled on `cyberdeck` (the `funnel=enabled` node) can use this.

```yaml
metadata:
  annotations:
    tailscale.com/hostname: "phantomcybersolutions"
    tailscale.com/funnel: "true"
```

Used by `phantom-site` to serve phantomcybersolutions.com publicly.

## Image Registry

All workload images live at `ghcr.io/realphantomlee/*`. Multi-arch images (linux/amd64 + linux/arm64) are required for any workload that might schedule to a Pi node (aipi, cyberdeck). The standard build command:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  --tag ghcr.io/realphantomlee/<name>:latest --push .
```

A GHCR pull secret (`ghcr-pull`) is created imperatively in each namespace that needs to pull private images.
