# seokpan-gitops

석판팀 「石나가는 판단」 1차 프로젝트의 Kubernetes Desired State와 Argo CD 배포 구성을 관리하는 Repository입니다.

## Repository Structure

```text
seokpan-gitops/
├── apps/
├── platform/
├── .gitignore
└── README.md
```

## Responsibility

이 Repository는 Kubernetes Cluster 위에서 Argo CD가 지속적으로 동기화할 Desired State를 관리합니다.

### apps

실제 서비스 애플리케이션 Runtime에 해당하는 Kubernetes 리소스를 관리합니다.

주요 대상은 다음과 같습니다.

* Frontend
* Backend
* Redis
* 애플리케이션별 Service
* 애플리케이션별 HPA 및 NetworkPolicy

세부 디렉터리 구조와 Manifest 구성 방식은 실제 배포 구현 착수 시 확정합니다.

### platform

Kubernetes 위에서 동작하는 공통 플랫폼 구성요소의 Desired State를 관리합니다.

주요 대상은 다음과 같습니다.

* Gateway API 관련 구성
* Metrics Server
* Prometheus
* Grafana
* Loki
* Grafana Alloy
* Alertmanager
* 공통 플랫폼 NetworkPolicy 및 관련 Kubernetes 리소스

세부 디렉터리 구조는 각 플랫폼 구성요소 구현 착수 시 확정합니다.

## Repository Boundary

애플리케이션 소스코드는 `seokpan-app`에서 관리합니다.

Host·VM·Network·Kubernetes Bootstrap 및 외부 인프라 자동화는 `seokpan-infra`에서 관리합니다.

이 Repository는 Argo CD가 지속적으로 관리하는 Kubernetes Desired State의 Source of Truth 역할을 담당합니다.

## Security

Password, Token, Private Key, kubeconfig Credential 및 실제 Kubernetes Secret 값은 Repository에 저장하지 않습니다.

