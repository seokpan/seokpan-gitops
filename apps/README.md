# Applications

「石나가는 판단」 서비스 애플리케이션의 Kubernetes Desired State를 관리하는 영역입니다.

Frontend와 Backend 등 실제 서비스 Application Runtime과 관련된 Kubernetes 리소스를 이 디렉터리 아래에서 관리합니다.

현재 구조:

```text
apps/
├── backend/
└── frontend/
```

- `apps/backend/`: Backend Deployment·Service·Runtime 설정과 Image 갱신 기준
- `apps/frontend/`: Frontend Deployment·Service와 Image 갱신 기준

두 경로 모두 현재는 Runtime 활성화 전 기반 단계입니다. 실제 Container Image와 필요한 Provider가 준비되기 전에는 `replicas: 0`과 `git-pending` sentinel을 유지하며, Argo CD Child Application도 연결하지 않습니다.

Argo CD Child `Application` CR 선언은 `argocd/applications/`에서 관리합니다. Redis 등 공통 Runtime Platform 구성은 `platform/`에서 관리합니다.

실제 Runtime 활성화는 Container Image Digest, Provider 연결, 필요한 Secret, Smoke Test 준비 상태를 확인한 뒤 별도 Issue/PR에서 진행합니다.

관련: `seokpan-gitops#29`
