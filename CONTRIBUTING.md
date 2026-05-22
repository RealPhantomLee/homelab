# Contributing

How to add or modify workloads in this repo. Keep it short — the existing apps are the best reference.

## Adding a New Workload

1. **Create the manifest directory** under `apps/<workload>/`:

   ```
   apps/<workload>/
     namespace.yaml
     deployment.yaml
     service.yaml
     pvc.yaml           (only if stateful)
     kustomization.yaml
   ```

2. **Use the existing apps as templates.** `apps/phantom-site/` is the simplest (stateless, Funnel-exposed); `apps/vaultkeeper/` is the canonical stateful example (PVCs, node-pinned).

3. **Pin to a node** if the workload is stateful or has arch-specific requirements:

   ```yaml
   spec:
     template:
       spec:
         nodeSelector:
           kubernetes.io/hostname: aipi   # or blacknode / cyberdeck
   ```

4. **Build multi-arch images** if the workload may schedule on a Pi node:

   ```bash
   docker buildx build --platform linux/amd64,linux/arm64 \
     --tag ghcr.io/realphantomlee/<name>:latest --push .
   ```

5. **Create the ArgoCD Application** under `clusters/homelab/apps/<workload>.yaml`. Copy `clusters/homelab/apps/phantom-site.yaml` and change `metadata.name`, `spec.source.path`, and `spec.destination.namespace`.

6. **Commit and push.** The root Application picks up the new child Application on next sync (within ~3 minutes, or trigger manually via `argocd app sync root`).

## Kustomization Patterns

Every `apps/<workload>/kustomization.yaml` follows the same shape:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
metadata:
  name: <workload>

namespace: <workload>

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  # - pvc.yaml         (if stateful)

commonLabels:
  app: <workload>
  managed-by: argocd
```

For workloads with sub-components (see `apps/local-ai/` — anythingllm, homarr, ollama), the parent `kustomization.yaml` lists subdirectories instead of individual files:

```yaml
resources:
  - namespace.yaml
  - anythingllm/
  - homarr/
  - ollama/
```

Each subdirectory has its own `kustomization.yaml`.

### Service Exposure Choices

Pick the right pattern (see [ARCHITECTURE.md](ARCHITECTURE.md#service-exposure-patterns) for details):

| Goal                       | Service spec                                                                         |
|----------------------------|--------------------------------------------------------------------------------------|
| Internal cluster only      | `type: ClusterIP`                                                                    |
| Tailnet-only access        | `type: LoadBalancer` + `loadBalancerClass: tailscale` + `tailscale.com/hostname`     |
| Public internet (Funnel)   | All of the above + `tailscale.com/funnel: "true"`, scheduled on cyberdeck            |

### Secrets

Never commit Secrets. Reference them via `secretKeyRef`/`envFrom`, and create them imperatively:

```bash
kubectl create secret generic <name> \
  --namespace <ns> \
  --from-literal=KEY=value
```

Document the required Secrets in a comment at the top of the manifest that references them — see `apps/local-ai/homarr/deployment.yaml` for an example.

## Testing Changes Locally

### Validate Kustomize output

```bash
kubectl kustomize apps/<workload>
```

This renders the final YAML without applying it. Use this to sanity-check labels, namespaces, and image tags.

### Dry-run against the live cluster

```bash
kubectl kustomize apps/<workload> | kubectl apply --dry-run=server -f -
```

Server-side dry-run catches schema violations, RBAC issues, and admission webhook rejections without changing cluster state.

### Diff against the live cluster

```bash
kubectl kustomize apps/<workload> | kubectl diff -f -
```

Shows what would change if ArgoCD synced the manifests right now. Useful for catching unintended drift before pushing.

### ArgoCD CLI diff

```bash
argocd app diff <workload>
```

The same diff but from ArgoCD's perspective (accounts for Kustomize options configured on the Application).

## Pull Request Checklist

- [ ] `kubectl kustomize apps/<workload>` renders without errors
- [ ] Image is multi-arch if scheduling on a Pi node
- [ ] No Secrets, OAuth tokens, or `.env` values are committed
- [ ] Node pinning (`nodeSelector`) matches the storage / arch requirements
- [ ] If adding a new Application, the root Application's `clusters/homelab/apps/` directory contains the new manifest
- [ ] README workload table is updated if the workload is new or changes status
