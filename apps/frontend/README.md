# Frontend Runtime Desired State

이 디렉터리는 `application` Namespace의 Frontend Kubernetes Desired State 기반을 관리합니다.

## 현재 단계

아직 실제 Runtime 활성화 단계가 아닙니다.

- Deployment는 `replicas: 0`으로 유지합니다.
- Image는 `git-pending` sentinel을 사용합니다.
- Argo CD Child Application은 아직 연결하지 않습니다.
- 실제 Frontend 서비스 UI와 Container Image가 준비된 뒤 별도 PR에서 활성화합니다.

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
6. Argo CD Child Application 연결 검토
7. Gateway `/` 경로 연결 검토

## 후속 연결

- `seokpan-gitops#27`, `seokpan-app#40` — Jenkins → Harbor → GitOps PR
- `seokpan-gitops#29` — 현재 작업
