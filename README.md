# Backstage Kubernetes Manifests

Backstage 플랫폼과 관련 인프라의 GitOps 매니페스트 레포지토리입니다.

## Architecture

ArgoCD가 이 레포지토리를 감시하여 Kubernetes 클러스터에 자동 배포합니다.

## Repository Structure

```text
k8s/
├── bootstrap/
│   └── argocd/
│       ├── apps/         # ArgoCD Application manifests
│       ├── appsets/      # ArgoCD ApplicationSet manifests
│       ├── ingress/      # ArgoCD access manifests
│       └── kustomization.yaml
├── platform/
│   ├── backstage/
│   │   ├── base/         # Backstage workloads, services, config, postgres
│   │   └── overlays/prod/
│   ├── devlake/base/     # Namespace and supporting resources
│   └── monitoring/base/  # Namespace and supporting resources
├── infra/
│   ├── ingress-nginx/base/
│   └── metallb/base/
└── kustomization.yaml
```

## Related Repositories

- **Application Source**: [backstage-app](https://github.com/backstage-test-1/backstage-app)

## GitOps Workflow

1. backstage-app의 CI가 새 이미지를 빌드하고 GHCR에 푸시
2. CI가 이 레포의 `platform/backstage/base/deployment-backend.yaml`과 `platform/backstage/base/deployment-frontend.yaml`의 이미지 태그를 자동 업데이트
3. ArgoCD가 변경사항을 감지하고 클러스터에 자동 배포

## Security Notes

⚠️ **시크릿 파일은 Git에 포함되지 않습니다:**
- `platform/backstage/base/secret-backstage.yaml`
- `platform/backstage/base/database/secret-postgres.yaml`
- `platform/backstage/base/notifications/secret-argocd-notifications.yaml`
- `monitoring` namespace `loki-minio-credentials` (manual secret, not committed)
- `backstage` namespace `postgres-backup-minio-credentials` (manual secret, not committed)

이 파일들은 클러스터에 수동으로 `kubectl apply` 해야 합니다.

## Manual Secret Deployment

```bash
# Backstage secrets
kubectl apply -f platform/backstage/base/secret-backstage.yaml

# PostgreSQL secrets  
kubectl apply -f platform/backstage/base/database/secret-postgres.yaml

# ArgoCD notifications secrets
kubectl apply -f platform/backstage/base/notifications/secret-argocd-notifications.yaml

# MinIO credentials for Loki (create a local manifest first; do not commit credentials)
kubectl apply -f /path/to/loki-minio-credentials.yaml

# MinIO credentials for PostgreSQL backups (create a local manifest first; do not commit credentials)
kubectl apply -f /path/to/postgres-backup-minio-credentials.yaml
```

## MinIO rollout manual secret shape

Create the following secrets outside Git before syncing the rollout:

### `monitoring/loki-minio-credentials`
- `MINIO_ENDPOINT` as `host:port`
- `MINIO_ACCESS_KEY_ID`
- `MINIO_SECRET_ACCESS_KEY`
- `MINIO_LOKI_BUCKET`

### `backstage/postgres-backup-minio-credentials`
- `MINIO_ENDPOINT` as `host:port`
- `MINIO_ACCESS_KEY_ID`
- `MINIO_SECRET_ACCESS_KEY`
- `MINIO_BACKUP_BUCKET`
- optional: `MINIO_BACKUP_PREFIX` (defaults to `backstage-postgres`)

Use the **same `MINIO_ACCESS_KEY_ID` / `MINIO_SECRET_ACCESS_KEY` pair** in both secrets. Only the bucket or prefix should differ between Loki and PostgreSQL backups.

The Loki migration manifest keeps the new TSDB schema disabled behind a `2099-01-01` sentinel cutover date. After the Loki 2.9.3 upgrade is verified, replace that date with the real **future UTC cutover date** and switch `compactor.shared_store` from `filesystem` to `s3` before syncing Phase C.

Operational note: a live check on 2026-04-03 showed Loki `/data` at roughly `5.6G / 45G` used on the backing NFS export, but `df -h /data` should still be part of the pre-cutover checklist because TSDB cache/active index files remain on the same mounted path.

## MetalLB and ingress-nginx LoadBalancer

Install MetalLB with the upstream native manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```

Then apply the repository-managed MetalLB and ingress-nginx configuration:

```bash
kubectl apply -k infra/metallb/base
kubectl apply -k infra/ingress-nginx/base
```

Before applying the IPAddressPool, replace the example address range with IPs that are actually reserved and free on the node subnet.

## ArgoCD Setup

```bash
# Add repository to ArgoCD
argocd repo add https://github.com/backstage-test-1/backstage-manifests

# Bootstrap ArgoCD-managed applications
kubectl apply -k bootstrap/argocd
```
