# Setup

Step-by-step bring-up of the homelab cluster from scratch. Estimated time: 60–90 minutes if everything goes smoothly.

## Prerequisites

- Three machines on the same Tailscale tailnet:
  - `blacknode` (x86_64, control-plane)
  - `aipi` (Pi 5, arm64 worker)
  - `cyberdeck` (Pi, arm64 worker, Funnel-enabled)
- A Tailscale account with admin access (for OAuth + Funnel ACLs)
- A GitHub account with write access to `RealPhantomLee/homelab` (for ArgoCD repo sync) and `ghcr.io/realphantomlee/*` (for image pulls)
- On your workstation:
  - `kubectl` v1.28+
  - `helm` v3.x
  - `tailscale` CLI (already authenticated against the tailnet)

Verify each node is reachable by Tailscale IP from your workstation before starting:

```bash
tailscale status
ping 100.90.234.21   # blacknode
ping 100.76.179.112  # aipi
ping 100.121.96.1    # cyberdeck
```

## 1. Tailscale OAuth Setup

The Tailscale Operator needs OAuth credentials to provision tailnet devices for LoadBalancer services.

1. Open the Tailscale admin console: <https://login.tailscale.com/admin/settings/oauth>
2. Click **Generate OAuth client**.
3. Grant scopes: `Devices: Core` (read/write) and `Auth Keys` (write).
4. Add a tag (e.g. `tag:k8s`). The operator will tag the devices it creates with this tag.
5. Save the Client ID and Client Secret somewhere safe — you cannot view the secret again.
6. In **Access Controls** (ACLs), make sure `tag:k8s` exists and is owned by your user or an autogroup that includes the OAuth client. Example snippet:

   ```json
   "tagOwners": {
     "tag:k8s": ["autogroup:admin"]
   }
   ```

7. Enable Funnel for the `cyberdeck` host in the ACL `nodeAttrs` section (Tailscale requires this before Funnel works):

   ```json
   "nodeAttrs": [
     { "target": ["cyberdeck"], "attr": ["funnel"] }
   ]
   ```

Paste the Client ID and Client Secret into `bootstrap/tailscale-operator/values.yaml` (`oauth.clientId` and `oauth.clientSecret`). **Do not commit this file with real values** — either revert it before pushing or use Sealed Secrets / External Secrets.

## 2. GHCR Personal Access Token

Workload images live in private GHCR repos under `ghcr.io/realphantomlee/`. Each namespace that pulls a private image needs a pull secret.

1. Open GitHub: <https://github.com/settings/tokens?type=beta> (or classic tokens).
2. Generate a fine-grained PAT with **Packages: read** scope, limited to the `realphantomlee` org/user.
3. Copy the token (`ghp_...`).

Create the pull secret in each workload namespace (run after the namespaces exist — ArgoCD creates them on first sync):

```bash
for ns in phantom-site vaultkeeper local-ai; do
  kubectl create secret docker-registry ghcr-pull \
    --namespace "$ns" \
    --docker-server=ghcr.io \
    --docker-username=realphantomlee \
    --docker-password='<YOUR_PAT_HERE>' \
    --docker-email=chucktuck22@gmail.com
done
```

If a workload's Deployment doesn't already reference `imagePullSecrets`, add it under the pod `spec`:

```yaml
spec:
  imagePullSecrets:
    - name: ghcr-pull
```

## 3. Install k3s

### On blacknode (control-plane)

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --flannel-iface=tailscale0 \
  --node-ip=100.90.234.21 \
  --advertise-address=100.90.234.21 \
  --tls-san=100.90.234.21 \
  --write-kubeconfig-mode=644 \
  --disable=traefik
```

Notes:
- `--disable=traefik` because we use the Tailscale Operator for ingress, not Traefik.
- `local-path` StorageClass is included by default in k3s — no extra install needed.

Grab the node token and kubeconfig:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token   # save as $K3S_TOKEN
sudo cat /etc/rancher/k3s/k3s.yaml                # copy to workstation, replace 127.0.0.1 with 100.90.234.21
```

### On aipi (Pi 5 worker)

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://100.90.234.21:6443 \
  K3S_TOKEN=<token from blacknode> sh -s - agent \
  --flannel-iface=tailscale0 \
  --node-ip=100.76.179.112
```

### On cyberdeck (Pi worker, Funnel-enabled)

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://100.90.234.21:6443 \
  K3S_TOKEN=<token from blacknode> sh -s - agent \
  --flannel-iface=tailscale0 \
  --node-ip=100.121.96.1 \
  --node-label=funnel=enabled
```

The `funnel=enabled` label lets the Tailscale Operator know this is the only node permitted to terminate public Funnel traffic.

### Verify

From your workstation (with kubeconfig in place):

```bash
kubectl get nodes -o wide
# Expect: blacknode (Ready, control-plane), aipi (Ready), cyberdeck (Ready)
```

## 4. Install the Tailscale Operator

```bash
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
helm install tailscale-operator tailscale/tailscale-operator \
  --namespace tailscale --create-namespace \
  -f bootstrap/tailscale-operator/values.yaml
```

Verify:

```bash
kubectl get pods -n tailscale
kubectl logs -n tailscale -l app=tailscale-operator
```

The operator should appear as a device named `k8s-operator` (per `operatorConfig.hostname` in values.yaml) in your Tailscale admin console.

## 5. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the rollout:

```bash
kubectl wait --for=condition=available --timeout=300s \
  deployment/argocd-server -n argocd
```

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

(Optional) Expose the ArgoCD UI on the tailnet via the Tailscale Operator:

```bash
kubectl annotate svc argocd-server -n argocd \
  tailscale.com/hostname=argocd
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer", "loadBalancerClass": "tailscale"}}'
```

## 6. Bootstrap the Root Application

This is the moment GitOps takes over. Apply the root Application — ArgoCD reads `clusters/homelab/apps/` from the repo and syncs every child Application automatically.

```bash
kubectl apply -n argocd -f clusters/homelab/apps/root-app.yaml
```

Watch it converge:

```bash
kubectl get applications -n argocd -w
```

Expect to see: `root`, `phantom-site`, `vaultkeeper`, `local-ai`, `monitoring`. They should reach `Synced / Healthy` (or `OutOfSync` for the workloads with `replicas: 0` and missing images — see [README.md workloads table](README.md#workloads)).

## 7. Secrets

Some workloads need manually-created Secrets that the manifests reference but the repo does not contain:

```bash
# Homarr encryption key (must be 64-char hex)
kubectl create secret generic homarr-secrets \
  --namespace local-ai \
  --from-literal=SECRET_ENCRYPTION_KEY=$(openssl rand -hex 32)

# GHCR pull secrets (see section 2 above for all namespaces)
```

## Troubleshooting

### Nodes won't join: `dial tcp 100.90.234.21:6443: i/o timeout`

The agent can't reach the control plane over the tailnet. Verify:

- `tailscale status` on all three nodes shows them online and able to ping each other.
- Tailscale ACLs allow port `6443/tcp` between nodes.
- k3s server on blacknode is listening on `100.90.234.21:6443`: `sudo ss -tlnp | grep 6443`.

### Pods stuck in `ImagePullBackOff`

- For `ghcr.io/realphantomlee/*` images: the namespace is missing the `ghcr-pull` secret, or the Deployment doesn't reference it. See section 2.
- The image doesn't exist yet (e.g. `anythingllm`, `homarr`, `vaultkeeper-server`). These Deployments have `replicas: 0` on purpose until their images are built and pushed. Check the comment at the top of each `deployment.yaml` for the disabled-reason.

### `exec format error` on a Pi node

The image is `linux/amd64`-only. Rebuild as multi-arch:

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  --tag ghcr.io/realphantomlee/<name>:latest --push .
```

### Tailscale Operator pod is `CrashLoopBackOff`

- OAuth credentials are wrong or have insufficient scopes (need `Devices` read/write).
- `tag:k8s` is not in the tailnet ACL `tagOwners`. The operator cannot create devices with an unowned tag.
- Currently `bootstrap/tailscale-operator/values.yaml` has `operatorDeploy.replicas: 0` due to a prior ACL issue — set it back to `1` when ready.

### ArgoCD app shows `OutOfSync` but never syncs

- Check the Application's status: `kubectl describe application <name> -n argocd`.
- `selfHeal` only triggers on drift detection — it can lag by up to 3 minutes. Force a sync: `argocd app sync <name>` (or click Sync in the UI).
- Repo URL case-sensitivity: ArgoCD treats `github.com/realphantomlee/homelab` and `github.com/RealPhantomLee/homelab` as different repos. The root app uses the capitalized form; child apps currently use lowercase. Normalize if you see auth issues.

### PVC stuck in `Pending`

- `local-path` provisions lazily — the PVC only binds once a pod is scheduled. If the pod is also Pending, check `kubectl describe pod <pod>` for the scheduling failure (usually a `nodeSelector` mismatch).
- Verify the target node has disk space: `ssh <node> df -h /var/lib/rancher/k3s/storage`.

### `phantomcybersolutions.com` returns 502 via Funnel

- Verify the `phantom-site` pod is running on cyberdeck: `kubectl get pods -n phantom-site -o wide`.
- Verify Funnel is provisioned: look in the Tailscale admin console for a device named `phantomcybersolutions`.
- Check operator logs: `kubectl logs -n tailscale -l app=tailscale-operator`.
