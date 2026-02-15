# Backstage Kubernetes Manifests

Backstage 애플리케이션의 Kubernetes 배포 매니페스트 레포지토리입니다.

## Architecture

ArgoCD가 이 레포지토리를 감시하여 Kubernetes 클러스터에 자동 배포합니다.

## Repository Structure

```
backstage/
├── namespace.yaml              # backstage 네임스페이스
├── 00-backstage-secrets.yaml   # Backstage 시크릿 (Git에 미포함)
├── 01-backstage.yaml           # Backstage Deployment
├── 02-backstage-service.yaml   # Backstage Service
├── 04-config.yaml              # Backstage ConfigMap
└── database/
    ├── 01-postgres-secret.yaml  # PostgreSQL 시크릿 (Git에 미포함)
    ├── 02-postgres-storage.yaml # PV/PVC
    ├── 03-postgres.yaml         # PostgreSQL Deployment
    └── 04-postgres-service.yaml # PostgreSQL Service

argocd/
└── backstage-application.yaml  # ArgoCD Application CR
```

## Related Repositories

- **Application Source**: [backstage-app](https://github.com/backstage-test-1/backstage-app)

## GitOps Workflow

1. backstage-app의 CI가 새 이미지를 빌드하고 GHCR에 푸시
2. CI가 이 레포의 `01-backstage.yaml`의 이미지 태그를 자동 업데이트
3. ArgoCD가 변경사항을 감지하고 클러스터에 자동 배포

## Security Notes

⚠️ **시크릿 파일은 Git에 포함되지 않습니다:**
- `00-backstage-secrets.yaml`
- `database/01-postgres-secret.yaml`

이 파일들은 클러스터에 수동으로 `kubectl apply` 해야 합니다.

## Manual Secret Deployment

```bash
# Backstage secrets
kubectl apply -f backstage/00-backstage-secrets.yaml

# PostgreSQL secrets  
kubectl apply -f backstage/database/01-postgres-secret.yaml
```

## ArgoCD Setup

```bash
# Add repository to ArgoCD
argocd repo add https://github.com/backstage-test-1/backstage-manifests

# Deploy application
kubectl apply -f argocd/backstage-application.yaml
```
