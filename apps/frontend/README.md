# Frontend Runtime Desired State

이 디렉터리는 `application` Namespace의 Frontend Kubernetes Desired State 기반을 관리합니다.

## 현재 단계

아직 실제 Runtime 활성화 단계가 아닙니다.

- Deployment는 `replicas: 0`으로 유지합니다.
- Image는 `git-pending` sentinel을 사용합니다.
- Argo CD Child Application `apps-frontend`는 Root Application에 편입되어 `apps/frontend`를 `main` 기준으로 관리합니다.
- 실제 Frontend Runtime은 별도 활성화 PR에서 진행합니다.

현재 GitOps 관리 편입과 실제 Frontend Runtime 활성화는 별개의 상태입니다. `apps-frontend` Application이 존재하고 Desired State가 Sync되어 있어도 `replicas: 0`, `git-pending` 상태에서는 실제 Frontend Pod가 실행되지 않습니다.

## Runtime 계약

- Deployment / Service: `frontend`
- Container / Service Port: `8080`, name `http`
- Image: `harbor.seokpan.soldesk.store/seokpan/frontend`
- Frontend는 환경별 Backend 절대 URL을 Image에 굽지 않고 같은 Origin의 `/api/v1`, `/ws/v1`을 사용합니다.

## Image 갱신 계약

CI는 `git-<main-commit-12자리>` Tag로 Harbor에 Push한 뒤 실제 Digest를 확인합니다.

첫 Runtime 활성화 시 `kustomization.yaml`의 `images` 항목을 실제 Digest 고정 방식으로 전환하고, 이후 Jenkins는 같은 Image 항목의 Digest만 변경하는 GitOps PR을 생성합니다.

`latest`는 사용하지 않습니다.

## Runtime 활성화 전 Gate

다음 조건을 확인한 뒤 별도 PR에서 Runtime을 활성화합니다.

1. 실제 Frontend 구현 완료
2. Frontend Container Image 존재
3. Harbor Push 및 실제 Digest 확인
4. SPA Fallback과 `/health/live` 등 Container Runtime 동작 확인
5. 초기 Runtime Smoke Test 준비
6. Argo CD Child Application 연결 — 완료
   - `apps-frontend`가 Root Application에 편입되어 `apps/frontend`를 `main` 기준으로 관리하며 `prune: true`, `selfHeal: true`를 사용합니다.
   - Runtime 활성화 전에는 `apps-frontend`의 Sync/Health와 실제 적용 Revision을 다시 확인합니다.
7. Gateway `/` 경로 연결 검토

## 후속 연결

- `seokpan-gitops#58` — Backend/Frontend Child Application Root 편입 및 Sync 검증
- `seokpan-gitops#27`, `seokpan-app#40` — Jenkins → Harbor → GitOps PR
- `seokpan-gitops#29` — Backend/Frontend Runtime Desired State 기반
