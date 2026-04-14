# Agent instructions for nrp-rustfs

## Overview

This repo manages a RustFS S3-compatible object storage deployment on NRP Nautilus in the `boettiger-lab` namespace. RustFS (https://rustfs.com) is a Rust-based S3 server, not MinIO. Do not use MinIO tools (`mc`, `mcli`) — use `rc` (the RustFS CLI) instead.

## Endpoints

- **S3 API:** `https://rustfs.nrp-nautilus.io`
- **Web Console:** `https://rustfs-console.nrp-nautilus.io`

## rc CLI reference

`rc` is the RustFS CLI (https://github.com/rustfs/cli). Install with `cargo install rustfs-cli`. It is NOT in apt repos.

### Alias setup

The admin alias must be configured before any operations:

```bash
ACCESS_KEY=$(kubectl -n boettiger-lab get secret rustfs-credentials \
  -o jsonpath='{.data.RUSTFS_ACCESS_KEY}' | base64 -d)
SECRET_KEY=$(kubectl -n boettiger-lab get secret rustfs-credentials \
  -o jsonpath='{.data.RUSTFS_SECRET_KEY}' | base64 -d)
rc alias set nrp https://rustfs.nrp-nautilus.io "$ACCESS_KEY" "$SECRET_KEY"
```

### S3 operations

```bash
rc mb nrp/<bucket>                          # Create bucket
rc rb nrp/<bucket>                          # Remove bucket
rc ls nrp/                                  # List buckets
rc ls nrp/<bucket>/                         # List objects
rc cp <src> nrp/<bucket>/<key>              # Upload
rc cp nrp/<bucket>/<key> <dest>             # Download
rc rm nrp/<bucket>/<key>                    # Delete object
```

### IAM user management

```bash
rc admin user add nrp/ <username> <password>
rc admin user list nrp/
rc admin user info nrp/ <username>
rc admin user remove nrp/ <username>
rc admin user enable nrp/ <username>
rc admin user disable nrp/ <username>
```

### IAM policy management

Policies use standard AWS IAM policy JSON format.

```bash
rc admin policy create nrp/ <policy-name> <policy-file.json>
rc admin policy list nrp/
rc admin policy info nrp/ <policy-name>
rc admin policy remove nrp/ <policy-name>
rc admin policy attach nrp/ <policy-name> --user <username>
rc admin policy detach nrp/ <policy-name> --user <username>
```

#### Read-only policy template

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:GetBucketLocation", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::<bucket>",
        "arn:aws:s3:::<bucket>/*"
      ]
    }
  ]
}
```

#### Read-write policy template

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:ListMultipartUploadParts",
        "s3:AbortMultipartUpload"
      ],
      "Resource": [
        "arn:aws:s3:::<bucket>",
        "arn:aws:s3:::<bucket>/*"
      ]
    }
  ]
}
```

### Service accounts (access keys)

Service accounts are access key pairs scoped to a parent user's permissions or a subset via an inline policy.

```bash
rc admin service-account create nrp/ <access-key> <secret-key>
rc admin service-account create nrp/ <access-key> <secret-key> --policy policy.json
rc admin service-account list nrp/
rc admin service-account info nrp/ <access-key>
rc admin service-account remove nrp/ <access-key>
```

### Group management

```bash
rc admin group add-members nrp/ <group> <user1> <user2>
rc admin group remove-members nrp/ <group> <user1>
rc admin group list nrp/
rc admin group info nrp/ <group>
```

## Admin REST API

RustFS exposes admin endpoints at `/rustfs/admin/v3/*` (JSON payloads, SigV4 auth). Prefer `rc` over direct API calls. Key endpoints:

- `PUT /rustfs/admin/v3/add-user?accessKey=<user>` — create user
- `GET /rustfs/admin/v3/list-users` — list users
- `PUT /rustfs/admin/v3/add-canned-policy?name=<policy>` — create policy
- `POST /rustfs/admin/v3/idp/builtin/policy/attach` — attach policy to user
- `GET /rustfs/admin/v3/info-canned-policy?name=<policy>` — get policy details

## Kubernetes

- **Namespace:** `boettiger-lab`
- **Credentials secret:** `rustfs-credentials` (keys: `RUSTFS_ACCESS_KEY`, `RUSTFS_SECRET_KEY`)
- **Storage:** 500 Gi PVC `rustfs-data` on `rook-ceph-block` (US West)
- **Deployment:** pinned to `us-west` region via node affinity
- **Ingress:** HAProxy with TLS termination at the ingress layer

### Redeploying

```bash
kubectl apply -f k8s/
kubectl -n boettiger-lab rollout status deployment/rustfs
```

### Health check

```bash
curl -sk https://rustfs.nrp-nautilus.io/health
```

## Important notes

- Do NOT use MinIO client (`mc`) or any MinIO tooling. Use `rc`.
- The AWS CLI works for S3 operations (`aws s3 --endpoint-url https://rustfs.nrp-nautilus.io`) but does NOT support IAM operations. Use `rc` for all user/policy management.
- `rc` subcommands use `list`, `remove`, `info` — not `ls`, `rm`, `desc` (those are aliases for S3 object operations only).
- Root credentials are in the `rustfs-credentials` K8s secret. Retrieve them via kubectl as shown above.
