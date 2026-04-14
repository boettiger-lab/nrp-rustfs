# nrp-rustfs

S3-compatible object storage on [NRP Nautilus](https://nrp.ai) using [RustFS](https://rustfs.com) ([GitHub](https://github.com/rustfs/rustfs)), backed by a 500 GB Ceph RBD persistent volume.

## Endpoints

| Service | URL |
|---------|-----|
| S3 API | `https://rustfs.nrp-nautilus.io` |
| Web Console | `https://rustfs-console.nrp-nautilus.io` |

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
```

TLS is terminated at the HAProxy ingress layer. Internal services use HTTP.

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

# 3. Verify
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
