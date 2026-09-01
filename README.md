# seokpan-gitops

석판팀 「石나가는 판단」 1차 프로젝트의 Kubernetes Desired State와 Argo CD 배포 구성을 관리하는 Repository입니다.

## Repository Structure

```text
seokpan-gitops/
├── argocd/
│   └── applications/
├── apps/
├── platform/
├── cicd/
├── observability/
├── .gitignore
└── README.md
```

## Responsibility

이 Repository는 Kubernetes Cluster 위에서 Argo CD가 지속적으로 동기화할 Desired State를 관리합니다.

### argocd/applications

Argo CD Root Application(App-of-Apps)이 관리할 Child `Application` CR 선언 전용 경로입니다.

각 선언은 실제 Workload Manifest를 직접 담지 않고 `apps/`, `platform/`, `cicd/`, `observability/` 아래의 Desired State 경로를 Argo CD에 연결합니다.

### apps

Frontend와 Backend 등 실제 서비스 Application Runtime에 해당하는 Kubernetes 리소스를 관리합니다.

애플리케이션별 Service, HPA, NetworkPolicy 등 Application에 직접 종속되는 Desired State도 이 영역에서 관리합니다.

### platform

Kubernetes 위에서 동작하는 공통 Runtime Platform 구성요소의 Desired State를 관리합니다.

주요 대상은 Gateway, Redis, Storage Client, Namespace/RBAC 등입니다.

### cicd

Jenkins 등 CI/CD Runtime Desired State를 관리합니다.

### observability

Prometheus, Grafana, Loki, Grafana Alloy, Alertmanager 등 관측 Runtime Desired State를 관리합니다.

## Repository Boundary

애플리케이션 소스코드는 `seokpan-app`에서 관리합니다.

Host·VM·Network·Kubernetes Bootstrap 및 외부 인프라 자동화는 `seokpan-infra`에서 관리합니다.

이 Repository는 Argo CD가 지속적으로 관리하는 Kubernetes Desired State의 Source of Truth 역할을 담당합니다.

## Security

Password, Token, Private Key, kubeconfig Credential 및 실제 Kubernetes Secret 값은 Repository에 저장하지 않습니다.

