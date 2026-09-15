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

Backend와 Frontend는 검증된 Image Digest를 사용하고 각각의 Argo CD Child Application으로 관리합니다. Backend Production Provider와 2 Replica 공유 상태 Gate를 통과했습니다. 이 Desired State는 Frontend와 같은 Origin Gateway Route의 활성화 대상을 선언하며, 실제 활성화 완료는 병합 후 Runtime Gate 통과로 판정합니다.

Argo CD Child `Application` CR 선언은 `argocd/applications/`에서 관리합니다. Redis 등 공통 Runtime Platform 구성은 `platform/`에서 관리합니다.

Runtime 활성화 완료는 Container Image Digest, Provider 연결, 필요한 Secret, Rollout과 실제 HTTP/WebSocket Smoke Test를 Cluster에서 확인한 뒤 기록합니다.

관련: `seokpan-gitops#29`
