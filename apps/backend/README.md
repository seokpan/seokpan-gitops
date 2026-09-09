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

Backend가 사용하는 공식 DB 주소는 `db.seokpan.soldesk.store:3306`이며, Runtime DB URL은 `backend-db-runtime` Secret의 `SEOKPAN_IDENTITY_DATABASE_URL`, `SEOKPAN_GAME_DATABASE_URL` Key를 통해 전달합니다.

공개 Root CA 전달 계약:

```text
ConfigMap: seokpan-internal-ca
Key:       ca.crt
Mount:     /etc/seokpan/pki/ca.crt
Env:       SEOKPAN_DATABASE_CA_FILE=/etc/seokpan/pki/ca.crt
```

`database-ca-configmap.yaml`에는 Ansible Controller의 공식 Root CA `/etc/pki/seokpan-ca/ca.crt`에서 인계받은 공개 인증서만 포함합니다.

2026-09-09 인계된 인증서는 기존 공개키·SKI를 유지하고, critical Key Usage에 Certificate Sign·CRL Sign을 포함합니다.
새 인증서의 X.509 SHA-256 Fingerprint는 다음과 같습니다.

```text
28:EE:82:23:2C:08:E7:48:A6:65:D7:98:65:AA:BB:5A:58:53:BE:32:BE:28:29:DB:37:F1:EE:02:6C:87:2F:60
```

이전 지문 `A3:3B:2F:BB:16:2B:41:5C:C7:91:7E:9B:F6:4A:6C:00:8E:CD:47:16:97:C4:F0:6D:0B:CF:DC:BA:E2:34:5B:96`은 교체 전 이력입니다.
현재 파일 대조에는 위 신규 지문을 사용합니다. Root CA 유효기간은 2026-09-09 08:03:30 UTC부터 2036-09-06 08:03:30 UTC까지이며, 서비스 인증서의 365일 발급 기준과 구분합니다.

인계·검증 근거는 [Infra PR #163](https://github.com/seokpan/seokpan-infra/pull/163)과
[GitOps #39](https://github.com/seokpan/seokpan-gitops/issues/39)에서 연결합니다.
2026-09-09 Ansible Controller의 OpenSSL 3.5.7에서 MaxScale 설정 파일과 동일한 공개 인증서 사본을
신규 CA로 `-x509_strict -purpose sslserver -verify_hostname db.seokpan.soldesk.store` 검증하여 통과했습니다(exit 0).
명령·두 인증서 지문·원본/사본 파일 해시·실행 결과는 [검증 완료 댓글](https://github.com/seokpan/seokpan-gitops/issues/39#issuecomment-5602090338)에 기록돼 있습니다.
공개 CA 파일의 지문·확장·자체 서명 검증과 MaxScale 서버 인증서 검증, 실제 DB 연결은 서로 다른 결과입니다.
Harbor API 시험 성공을 Backend DB 연결 성공으로 대신하지 않습니다.

ConfigMap 파일 변경만으로 실행 중인 Pod의 `subPath` 파일이 갱신되지는 않습니다.
실제 적용 전에 Backend와 Migration 실행 상태를 확인하고, 가동 중인 Backend는 승인된 재기동 후 파일 지문·TLS·서비스 상태를 확인합니다.
실행 중인 Migration은 임의로 중단하거나 재실행하지 않고 작업 종료와 CA 전환 시점을 조율합니다.
새 Migration Job도 실행 전에 신규 CA를 확인하며, CA 검증을 위해 Migration을 실행하지 않습니다.
Backend가 실제로 비활성 상태라면 CA 교체를 위해 Replica를 늘리거나 불필요한 재시작을 하지 않습니다.
자세한 실행·완료 기준은 [GitOps #39](https://github.com/seokpan/seokpan-gitops/issues/39)에서 관리하며 이 파일 변경만으로 해당 Issue를 종료하지 않습니다.

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
공급 자동화는 `seokpan-infra#150`에서 관리합니다.

승인형 One-shot Migration Job 실행 구조와 자산은 `seokpan-gitops#44`에서 관리합니다.
실행 자산은 `apps/backend/migration/`에 보관하며 일반 `apps/backend/kustomization.yaml`에는 포함하지 않아 Argo CD Auto-Sync 대상과 분리합니다.
실제 Migration 실행은 `seokpan-infra#150`의 Migration Secret 공급, 승인된 Backend Image Digest, DB 사전 Gate 완료 이후에만 수행합니다.

## Image 갱신 계약

CI는 `git-<main-commit-12자리>` Tag로 Harbor에 Push한 뒤 실제 Digest를 확인합니다.

첫 Runtime 활성화 시 `kustomization.yaml`의 `images` 항목을 실제 Digest 고정 방식으로 전환하고, 이후 Jenkins는 같은 Image 항목의 Digest만 변경하는 GitOps PR을 생성합니다.

`latest`는 사용하지 않습니다.

## Runtime 활성화 전 Gate

다음 조건을 확인한 뒤 별도 PR에서 Runtime을 활성화합니다.

1. Backend Container Image 존재
2. Harbor Push 및 실제 Digest 확인
3. `seokpan-app#50`의 TLS Client 구현 및 정적 검증 완료
4. Infra 인계값과 저장소 공개 CA Fingerprint 일치 확인. 실제 적용 후 ConfigMap·소비 Pod의 신규 지문 및 MaxScale TLS 결과는 별도 확인
5. MariaDB·Redis Provider 조립 완료
6. Runtime DB Secret 계약 확정 — 완료
   - Secret 공급 자동화·담당자 검증: [Infra #150](https://github.com/seokpan/seokpan-infra/issues/150) 완료. 실제 실행 전 Secret 준비 상태와 접근 권한은 다시 확인
7. One-shot Migration Kubernetes 실행 구조 확정 — 완료
   - 실제 실행: 승인된 Image Digest·DB 사전 Gate·실행 승인 대기. 실행 전 `seokpan-infra#150`의 Secret 공급 상태 확인
8. Provider 상태를 확인하는 readiness 기준 확인
9. 초기 `replicas: 1` Smoke Test 준비
10. Argo CD Child Application 연결 검토

1 Replica 통합 검증 전에는 2 Replica 이상으로 확장하지 않습니다.

## 후속 연결

- `seokpan-gitops#44` — 승인형 One-shot Migration Kubernetes Job 실행 구조
- `seokpan-gitops#35` — Backend MaxScale TLS 공개 CA 주입 구조
- `seokpan-app#50` — Backend/Alembic MaxScale TLS Client
- `seokpan-gitops#7` — Redis 실제 Backend 연결
- `seokpan-app#22` — Alembic Provider Gate
- `seokpan-infra#102` — MaxScale TLS Listener 완료
- `seokpan-infra#138` — TLS SAN 변경 감지 자동화 완료
- `seokpan-gitops#27`, `seokpan-app#40` — Jenkins → Harbor → GitOps PR
- `seokpan-gitops#29` — Backend/Frontend Runtime Desired State 기반
