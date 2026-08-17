# nrp-rustfs

S3-compatible object storage on [NRP Nautilus](https://nrp.ai) using [RustFS](https://rustfs.com) ([GitHub](https://github.com/rustfs/rustfs)), backed by a 500 GB Ceph RBD persistent volume.

## Endpoints

| Service | URL |
|---------|-----|
| S3 API | `https://rustfs.nrp-nautilus.io` |
| Web Console | `https://rustfs-console.nrp-nautilus.io` |

**From inside the cluster, use the Service, not the public URL:**
`http://rustfs-s3.boettiger-lab.svc.cluster.local`. It keeps traffic on the pod
network instead of sending it out through HAProxy and back — which for a large
upload is both slower and measurably less reliable.

⚠️ **Any in-cluster name must be listed in `RUSTFS_SERVER_DOMAINS`** (in
`k8s/deployment.yaml`) before a client can use it. rustfs decides addressing style
from the request `Host`: a Host on that allowlist — or a bare IP — is treated as
path style, and **anything else is parsed as virtual-host style, so its first label
is taken to be a bucket name.** An unlisted name therefore fails with a blanket
`AccessDenied` that looks like a credentials or policy problem and cannot be fixed
by changing either. The tell is that the denial covers `list_buckets` too, and that
the same key works against the ClusterIP (an IP cannot be virtual-host style). This
cost a day in geo-agent-ops#126; the five current spellings are all listed.

## Architecture

- **Namespace:** `boettiger-lab`
- **Storage:** 500 Gi PVC on `rook-ceph-block` (US West)
- **Region affinity:** pod pinned to `us-west` to co-locate with storage
- **Credentials:** stored in `rustfs-credentials` K8s secret
- **Image:** `rustfs/rustfs:latest`

## Kubernetes manifests

```
k8s/
  pvc.yaml              # 500 Gi PersistentVolumeClaim (rook-ceph-block)
  secret.yaml           # Secret template (do not commit real values)
  deployment.yaml       # RustFS deployment with health checks
  service-s3.yaml       # ClusterIP service for S3 API (port 80 -> 9000)
  service-console.yaml  # ClusterIP service for console (port 80 -> 9001)
  ingress-s3.yaml       # HAProxy ingress for S3 API
  ingress-console.yaml  # HAProxy ingress for console
  refresh-cronjob.yaml  # weekly age-reset of the Deployment (SA + Role + CronJob)
```

TLS is terminated at the HAProxy ingress layer. Internal services use HTTP.

### ⚠️ The Deployment is re-applied weekly from `main`

`refresh-cronjob.yaml` runs Sundays at 03:00 UTC and does `kubectl delete -f
k8s/deployment.yaml && kubectl apply -f k8s/deployment.yaml`, re-cloning this repo
each time. It exists because NRP culls Deployments on age and `rollout restart`
does not reset `creationTimestamp` — only a delete + re-create does.

**So `main` is the source of truth for the live Deployment, not the cluster.** A
change made only with `kubectl edit` or a local `kubectl apply` works for up to a
week and is then silently reverted the next Sunday. Commit and push it.

It applies only `deployment.yaml`; the Service, Ingress, and PVC are not age-culled
and are deliberately left alone (re-creating the PVC would destroy the data).

## Deploying from scratch

```bash
# 1. Create the secret with random credentials
kubectl -n boettiger-lab create secret generic rustfs-credentials \
  --from-literal=RUSTFS_ACCESS_KEY=$(openssl rand -hex 12) \
  --from-literal=RUSTFS_SECRET_KEY=$(openssl rand -hex 24)

# 2. Apply all manifests
kubectl apply -f k8s/pvc.yaml
kubectl apply -f k8s/service-s3.yaml -f k8s/service-console.yaml \
              -f k8s/ingress-s3.yaml -f k8s/ingress-console.yaml
kubectl apply -f k8s/deployment.yaml

# 3. Install the weekly age-reset so NRP's culler doesn't reap the Deployment
kubectl apply -f k8s/refresh-cronjob.yaml

# 4. Verify
kubectl -n boettiger-lab get pods -o wide
curl -sk https://rustfs.nrp-nautilus.io/health
```

## rc CLI

RustFS has its own CLI called `rc` for S3 and admin operations.

### Install

```bash
# Cargo
cargo install rustfs-cli

# Homebrew
brew install rustfs/tap/rc

# Binary releases
# https://github.com/rustfs/cli/releases
```

Not available in Ubuntu/Debian apt repos. See https://github.com/rustfs/cli for all options.

### Configure

```bash
# Retrieve credentials from K8s
ACCESS_KEY=$(kubectl -n boettiger-lab get secret rustfs-credentials \
  -o jsonpath='{.data.RUSTFS_ACCESS_KEY}' | base64 -d)
SECRET_KEY=$(kubectl -n boettiger-lab get secret rustfs-credentials \
  -o jsonpath='{.data.RUSTFS_SECRET_KEY}' | base64 -d)

rc alias set nrp https://rustfs.nrp-nautilus.io "$ACCESS_KEY" "$SECRET_KEY"
```

### Common operations

```bash
# Buckets
rc mb nrp/my-bucket
rc ls nrp/

# Upload / download
rc cp file.txt nrp/my-bucket/
rc cp nrp/my-bucket/file.txt ./

# IAM users
rc admin user add nrp/ username password
rc admin user list nrp/

# IAM policies
rc admin policy create nrp/ policy-name policy.json
rc admin policy attach nrp/ policy-name --user username
rc admin policy list nrp/
```

## Credential rotation

```bash
kubectl -n boettiger-lab delete secret rustfs-credentials
kubectl -n boettiger-lab create secret generic rustfs-credentials \
  --from-literal=RUSTFS_ACCESS_KEY=$(openssl rand -hex 12) \
  --from-literal=RUSTFS_SECRET_KEY=$(openssl rand -hex 24)
kubectl -n boettiger-lab rollout restart deployment/rustfs
```
