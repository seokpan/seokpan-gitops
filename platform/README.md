# Platform

Kubernetes 위에서 동작하는 공통 Runtime Platform 구성요소의 Desired State를 관리하는 영역입니다.

Gateway, Redis, Storage Client, Namespace/RBAC 등 Argo CD가 지속적으로 관리할 플랫폼 Kubernetes 리소스를 이 디렉터리 아래에서 관리합니다.

Argo CD Child `Application` CR 선언은 [`argocd/applications/`](../argocd/applications/)에서 관리합니다.

현재 구성은 이 디렉터리의 Manifest와 연결된 Child Application의 `source.path`를 기준으로 확인합니다. `platform/`은 Git의 파일 분류이며, 실제 배포 Namespace는 각 Manifest의 `metadata.namespace`와 Application의 `destination.namespace`를 확인합니다.

실제 적용·검증 범위와 미완료 항목은 [1차 종료 시점 상태](https://github.com/seokpan/seokpan-docs/blob/main/CURRENT_STATE.md) 및 각 구성요소의 기존 Issue/PR에서 추적합니다. 파일 존재나 Argo CD 관리 편입만으로 해당 구성요소의 전체 검증 완료를 판정하지 않습니다.
