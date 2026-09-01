# Applications

「石나가는 판단」 서비스 애플리케이션의 Kubernetes Desired State를 관리하는 영역입니다.

Frontend와 Backend 등 실제 서비스 Application Runtime과 관련된 Kubernetes 리소스를 이 디렉터리 아래에서 관리합니다.

Argo CD Child `Application` CR 선언은 `argocd/applications/`에서 관리하며, Redis 등 공통 Runtime Platform 구성은 `platform/`에서 관리합니다.

세부 디렉터리 구조와 Manifest 구성 방식은 실제 Application 배포 구현 착수 시 확정합니다.

