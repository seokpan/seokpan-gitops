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

MaxScale TLS Listener와 App Client 계약은 `seokpan-infra#102`, `seokpan-app#50` 기준으로 확정되어 있습니다.

Backend가 사용하는 공식 DB 주소는 `db.seokpan.soldesk.store:3306`이며, 실제 Runtime DB URL은 이후 확정되는 `SEOKPAN_IDENTITY_DATABASE_URL`, `SEOKPAN_GAME_DATABASE_URL` Secret 참조로 전달합니다.

공개 Root CA 전달 계약:

```text
ConfigMap: seokpan-internal-ca
Key:       ca.crt
Mount:     /etc/seokpan/pki/ca.crt
Env:       SEOKPAN_DATABASE_CA_FILE=/etc/seokpan/pki/ca.crt
```

`database-ca-configmap.yaml`에는 Ansible Controller의 공식 Root CA `/etc/pki/seokpan-ca/ca.crt`에서 가져온 공개 인증서만 포함합니다. GitOps 반영 전 `seokpan-app#50`에 기록된 X.509 SHA-256 Fingerprint와 일치함을 확인했습니다.

확인된 Fingerprint:

```text
A3:3B:2F:BB:16:2B:41:5C:C7:91:7E:9B:F6:4A:6C:00:8E:CD:47:16:97:C4:F0:6D:0B:CF:DC:BA:E2:34:5B:96
```

CA Private Key와 서비스 Private Key는 이 Repository에 포함하지 않습니다.

`SEOKPAN_MIGRATION_DATABASE_URL`과 `db_admin` Credential은 일반 Backend Deployment에 주입하지 않습니다.

DB Credential은 Runtime과 Migration의 권한 경계를 Kubernetes Secret에서도 분리합니다.

Runtime DB Secret:

    Secret: backend-db-runtime
    Namespace: application
    Consumer: Backend Deployment

    Keys:
    - SEOKPAN_IDENTITY_DATABASE_URL
    - SEOKPAN_GAME_DATABASE_URL

Migration DB Secret:

    Secret: backend-db-migration
    Namespace: application
    Consumer: 승인된 One-shot Migration Workload만

    Key:
    - SEOKPAN_MIGRATION_DATABASE_URL

세 DB URL은 모두 공식 Endpoint `db.seokpan.soldesk.store:3306`과 Database `stone_game`을 사용하며,
각각 `identity_svc`, `game_svc`, `db_admin`의 기존 계정 경계를 유지합니다.

실제 Password와 전체 DB URL은 Git에 저장하지 않습니다. GitOps는 Secret 이름·Key와 소비 Workload의
참조 계약만 관리합니다.

실제 Kubernetes Secret 값의 공급은 Ansible + Vault 방식을 사용하며,
필요한 Infra 자동화 변경은 `seokpan-infra`의 별도 Issue·Branch·PR에서 관리합니다.

One-shot Migration Workload의 구체적인 Resource 이름과 Kubernetes 실행 구조는 후속 GitOps 작업에서 확정합니다.

## Image 갱신 계약

CI는 `git-<main-commit-12자리>` Tag로 Harbor에 Push한 뒤 실제 Digest를 확인합니다.

첫 Runtime 활성화 시 `kustomization.yaml`의 `images` 항목을 실제 Digest 고정 방식으로 전환하고, 이후 Jenkins는 같은 Image 항목의 Digest만 변경하는 GitOps PR을 생성합니다.

`latest`는 사용하지 않습니다.

## Runtime 활성화 전 Gate

다음 조건을 확인한 뒤 별도 PR에서 Runtime을 활성화합니다.

1. Backend Container Image 존재
2. Harbor Push 및 실제 Digest 확인
3. `seokpan-app#50`의 TLS Client 구현 및 정적 검증 완료
4. Infra에서 인계한 Root CA와 GitOps `ca.crt` Fingerprint 일치 확인 — 완료
5. MariaDB·Redis Provider 조립 완료
6. 필요한 Runtime Secret 확정
7. 승인된 One-shot Migration 실행 준비
8. Provider 상태를 확인하는 readiness 기준 확인
9. 초기 `replicas: 1` Smoke Test 준비
10. Argo CD Child Application 연결 검토

1 Replica 통합 검증 전에는 2 Replica 이상으로 확장하지 않습니다.

## 후속 연결

- `seokpan-gitops#35` — Backend MaxScale TLS 공개 CA 주입 구조
- `seokpan-app#50` — Backend/Alembic MaxScale TLS Client
- `seokpan-gitops#7` — Redis 실제 Backend 연결
- `seokpan-app#22` — Alembic Provider Gate
- `seokpan-infra#102` — MaxScale TLS Listener 완료
- `seokpan-infra#138` — TLS SAN 변경 감지 자동화 완료
- `seokpan-gitops#27`, `seokpan-app#40` — Jenkins → Harbor → GitOps PR
- `seokpan-gitops#29` — Backend/Frontend Runtime Desired State 기반
