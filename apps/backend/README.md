# Backend Runtime Desired State

이 디렉터리는 `application` Namespace의 Backend Kubernetes Desired State 기반을 관리합니다.

## 현재 단계

아직 실제 Runtime 활성화 단계가 아닙니다.

- Deployment는 `replicas: 0`으로 유지합니다.
- Image는 `git-pending` sentinel을 사용합니다.
- Argo CD Child Application은 아직 연결하지 않습니다.
- 실제 DB URL 및 Credential은 포함하지 않습니다.

## Runtime 계약

- Deployment / Service: `backend`
- Container / Service Port: `8000`, name `http`
- Image: `harbor.seokpan.soldesk.store/seokpan/backend`
- Redis: `redis://redis.platform.svc.cluster.local:6379/0`
- Startup: `/health/startup`
- Liveness: `/health/live`
- Readiness: `/health/ready`
- `SEOKPAN_INSTANCE_ID`: Pod `metadata.name` Downward API

현재 `/health/ready`는 MariaDB·Redis Provider 상태까지 확인하지 않으므로 Provider readiness 완료와 구분합니다.

## DB / Migration 경계

일반 Backend Runtime은 이후 확정되는 `SEOKPAN_IDENTITY_DATABASE_URL`, `SEOKPAN_GAME_DATABASE_URL`만 사용합니다.

`seokpan-infra#102`의 MaxScale TLS/CA 기준이 확정되기 전까지 실제 DB URL Secret과 TLS Option을 이 디렉터리에 고정하지 않습니다.

`SEOKPAN_MIGRATION_DATABASE_URL`과 `db_admin` Credential은 일반 Backend Deployment에 주입하지 않습니다.

## Image 갱신 계약

CI는 `git-<main-commit-12자리>` Tag로 Harbor에 Push한 뒤 실제 Digest를 확인합니다.

첫 Runtime 활성화 시 `kustomization.yaml`의 `images` 항목을 실제 Digest 고정 방식으로 전환하고, 이후 Jenkins는 같은 Image 항목의 Digest만 변경하는 GitOps PR을 생성합니다.

`latest`는 사용하지 않습니다.

## Runtime 활성화 전 Gate

다음 조건을 확인한 뒤 별도 PR에서 Runtime을 활성화합니다.

1. Backend Container Image 존재
2. Harbor Push 및 실제 Digest 확인
3. MariaDB·Redis Provider 조립 완료
4. 필요한 Runtime Secret 확정
5. Provider 상태를 확인하는 readiness 기준 확인
6. 초기 `replicas: 1` Smoke Test 준비
7. Argo CD Child Application 연결 검토

1 Replica 통합 검증 전에는 2 Replica 이상으로 확장하지 않습니다.

## 후속 연결

- `seokpan-gitops#7` — Redis 실제 Backend 연결
- `seokpan-infra#102` — MaxScale TLS/CA
- `seokpan-app#22` — Alembic Provider Gate
- `seokpan-gitops#27`, `seokpan-app#40` — Jenkins → Harbor → GitOps PR
- `seokpan-gitops#29` — 현재 작업
