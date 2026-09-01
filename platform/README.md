# Platform

Kubernetes 위에서 동작하는 공통 Runtime Platform 구성요소의 Desired State를 관리하는 영역입니다.

Gateway, Redis, Storage Client, Namespace/RBAC 등 Argo CD가 지속적으로 관리할 플랫폼 Kubernetes 리소스를 이 디렉터리 아래에서 관리합니다.

Argo CD Child `Application` CR 선언은 `argocd/applications/`에서 관리합니다.

세부 디렉터리 구조와 Manifest 구성 방식은 각 플랫폼 구현 착수 시 확정합니다.

