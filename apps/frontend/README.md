# Frontend Runtime Desired State

이 디렉터리는 `application` Namespace의 Frontend Kubernetes Desired State 기반을 관리합니다.

## 현재 단계

Backend 2 Replica Production Provider Gate 통과 후 Frontend Runtime과 같은 Origin Gateway Route를 활성화합니다.

- Deployment는 `replicas: 2`이며 두 Worker에 강제 분산합니다.
- RollingUpdate는 `maxSurge: 0`, `maxUnavailable: 1`로 고정합니다.
- PDB `minAvailable: 1`은 Eviction API를 사용하는 계획된 축출에서 최소 한 Pod를 유지합니다.
- Image는 App A-10 변경을 포함해 검증한 Frontend Digest로 고정되어 있습니다.
- Deployment는 `application/harbor-pull-secret`을 명시적으로 참조합니다.
- Argo CD Child Application `apps-frontend`는 Root Application에 편입되어 `apps/frontend`를 `main` 기준으로 관리합니다.
- `/tmp`만 `emptyDir`로 제공하고 Root Filesystem은 읽기 전용으로 실행합니다.
- 종료 전 5초의 Endpoint 전파 여유를 두고 전체 종료 유예는 45초로 고정합니다.

## Runtime 계약

- Deployment / Service: `frontend`
- Container / Service Port: `8080`, name `http`
- Image: `harbor.seokpan.soldesk.store/seokpan/frontend`
- Frontend는 환경별 Backend 절대 URL을 Image에 굽지 않고 같은 Origin의 `/api/v1`, `/ws/v1`을 사용합니다.

## Image 갱신 계약

CI는 `git-<main-commit-12자리>` Tag로 Harbor에 Push한 뒤 실제 Digest를 확인합니다.

현재 `kustomization.yaml`의 `images` 항목은 App PR #75 병합 뒤 Jenkins main Image Build #5에서 검증한 실제 Digest로 고정되어 있습니다. 이후 Image 갱신 자동화 방식은 별도 설계하며, 승인된 GitOps PR에서 같은 Image 항목의 Digest를 변경합니다.

`latest`는 사용하지 않습니다.

## Runtime 활성화 Gate

이 활성화 변경은 다음 선행조건과 Render 결과를 확인한 뒤 병합하고, 병합 후 Runtime 검증으로 완료합니다.

1. 실제 Frontend 구현 완료 — 완료
2. Frontend Container Image 존재 — 완료
3. Harbor Push 및 실제 Digest 확인 — 완료
4. SPA Fallback과 `/health/live`를 포함한 Image Process Smoke — 완료
5. 초기 Runtime Smoke Test 준비 — 완료
6. Argo CD Child Application 연결 — 완료
   - `apps-frontend`가 Root Application에 편입되어 `apps/frontend`를 `main` 기준으로 관리하며 `prune: true`, `selfHeal: true`를 사용합니다.
   - 병합 후 `apps-frontend`의 Sync/Health와 실제 적용 Revision을 확인합니다.
7. Gateway의 현재 Client Route·Asset→Frontend, `/api/v1`·`/ws/v1`→Backend 연결 — 이 Desired State 변경에 포함, 실제 연결은 병합 후 확인

## 후속 연결

- `seokpan-gitops#58` — Backend/Frontend Child Application Root 편입 및 Sync 검증
- `seokpan-gitops#27`, `seokpan-app#40` — Jenkins → Harbor → GitOps PR
- `seokpan-gitops#29` — Backend/Frontend Runtime Desired State 기반
