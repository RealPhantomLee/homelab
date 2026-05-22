# Backup Strategy

This document outlines the backup approach for the homelab k3s cluster. None of
the tooling below is installed yet — this is reference material for when the
cluster carries data worth protecting (currently most apps are stateless or
re-deployable from this repo).

Recovery priority, highest to lowest:

1. **Cluster state** — etcd, secrets, ArgoCD applications.
2. **Application data** — VaultKeeper SQLite + encrypted blobs, Ollama models,
   Homarr config, AnythingLLM storage.
3. **Manifests** — already covered by this Git repository.

---

## 1. k3s etcd snapshots (cluster state)

k3s ships with built-in etcd snapshot support. Enable it on the server node:

```bash
# /etc/rancher/k3s/config.yaml
etcd-snapshot-schedule-cron: "0 */6 * * *"   # every 6 hours
etcd-snapshot-retention: 12                  # keep 3 days
etcd-snapshot-dir: /var/lib/rancher/k3s/server/db/snapshots
```

Restart k3s to apply: `sudo systemctl restart k3s`.

**Manual snapshot:**

```bash
sudo k3s etcd-snapshot save --name pre-upgrade
```

**Restore:**

```bash
sudo systemctl stop k3s
sudo k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=/var/lib/rancher/k3s/server/db/snapshots/<snapshot-name>
sudo systemctl start k3s
```

Ship snapshots off-node nightly via `rsync` to a Tailscale-reachable host (see
manual recipe below).

---

## 2. Velero (PVC + namespace backups)

[Velero](https://velero.io) is the recommended tool once the cluster has more
than one stateful workload. Install plan:

```bash
# Pick an S3-compatible target — MinIO on the LAN, Backblaze B2, or Cloudflare R2.
# Create a bucket called "homelab-velero" first, then:

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.9.0 \
  --bucket homelab-velero \
  --backup-location-config region=us-east-1,s3ForcePathStyle=true,s3Url=https://<endpoint> \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --uploader-type kopia
```

`--use-node-agent` + `--uploader-type kopia` is required because k3s default
storage is `local-path` (no CSI snapshots). Kopia performs file-level backup of
the PVC contents through a node-agent pod.

**Scheduled backups:**

```bash
velero schedule create daily-all \
  --schedule "0 3 * * *" \
  --ttl 168h \
  --include-namespaces local-ai,vaultkeeper,phantom-site

velero schedule create weekly-full \
  --schedule "0 4 * * 0" \
  --ttl 720h
```

**Restore a namespace:**

```bash
velero restore create --from-backup daily-all-20260101030000 \
  --include-namespaces vaultkeeper
```

---

## 3. PVC backup for local-path storage

The default k3s `local-path` provisioner stores PVC data under
`/var/lib/rancher/k3s/storage/<pvc-uid>` on the node where the pod is scheduled.
Without a CSI driver there are no volume snapshots, so backups are either:

- **Application-aware** — dump the data through the app (e.g.
  `sqlite3 vault.db .backup vault.db.bak` for VaultKeeper) before copying.
- **Filesystem-level** — quiesce the pod (`kubectl scale --replicas=0`), tar the
  PVC directory, then scale back up.

Stick with Velero+Kopia for the long term — the manual recipe below is the
fallback when Velero isn't installed yet.

---

## 4. Manual `tar` + `rsync` recipe (VaultKeeper fallback)

Run on the cluster node, scheduled by `cron` or `systemd-timer`. Replace
`<pvc-uid>` with the actual directory name (`kubectl -n vaultkeeper get pvc
vaultkeeper-data -o jsonpath='{.spec.volumeName}'`).

```bash
#!/usr/bin/env bash
set -euo pipefail

NS=vaultkeeper
PVC=vaultkeeper-data
BACKUP_HOST=aipi               # Tailscale hostname of backup target
BACKUP_DIR=/srv/backups/homelab/vaultkeeper
STAMP=$(date +%Y%m%d-%H%M%S)

# Resolve on-disk path for the PVC.
PV_NAME=$(kubectl -n "$NS" get pvc "$PVC" -o jsonpath='{.spec.volumeName}')
SRC="/var/lib/rancher/k3s/storage/${PV_NAME}_${NS}_${PVC}"

# Quiesce the deployment so SQLite isn't mid-write.
kubectl -n "$NS" scale deploy/vaultkeeper --replicas=0
kubectl -n "$NS" wait --for=delete pod -l app=vaultkeeper --timeout=60s || true

# Snapshot to a local tarball.
TARBALL="/tmp/${PVC}-${STAMP}.tar.zst"
sudo tar --zstd -cf "$TARBALL" -C "$(dirname "$SRC")" "$(basename "$SRC")"

# Ship off-node.
rsync -avh --remove-source-files "$TARBALL" "${BACKUP_HOST}:${BACKUP_DIR}/"

# Restart the app.
kubectl -n "$NS" scale deploy/vaultkeeper --replicas=1
```

**Retention on the backup host:**

```bash
# Keep last 14 days
find /srv/backups/homelab/vaultkeeper -name '*.tar.zst' -mtime +14 -delete
```

**Restore:**

```bash
kubectl -n vaultkeeper scale deploy/vaultkeeper --replicas=0
sudo tar --zstd -xf vaultkeeper-data-<stamp>.tar.zst \
  -C /var/lib/rancher/k3s/storage/
kubectl -n vaultkeeper scale deploy/vaultkeeper --replicas=1
```

---

## What is NOT backed up by any of the above

- **Imperatively-created Secrets** (`homarr-secrets`,
  `tailscale-oauth-secret`, etc.). Keep the originating values in a password
  manager — they are intentionally not stored in Git or etcd snapshots in
  plaintext form. Velero will capture them encrypted at rest in object
  storage; consider Sealed Secrets or External Secrets long term.
- **Ollama model blobs** — large and re-downloadable. Skip them unless
  bandwidth is constrained; `ollama pull` rebuilds the cache.
- **Node OS state** — handled separately by host-level Arch backups, not by
  this cluster strategy.
