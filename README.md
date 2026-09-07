# seokpan-gitops

석판팀 1차 프로젝트의 **Kubernetes Desired State와 Argo CD 기반 배포 구성을 관리하는 Repository**입니다.

Kubernetes Cluster에서 실행되는 Application, Platform, CI/CD, Observability 구성의 Desired State를 Git으로 관리하고 **Argo CD를 통해 Cluster Runtime과 지속적으로 동기화**합니다.

---

## Architecture

프로젝트 전체 Infrastructure와 서비스 구성은 다음 구조를 기준으로 합니다.

```text
                    seokpan-infra
                         │
              Kubernetes Cluster Bootstrap
                         │
                         ▼
                 Kubernetes Cluster
                         │
                    Argo CD Root
                         │
                         ▼
              argocd/applications/
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
        apps/         platform/       cicd/
          │              │              │
     Frontend/Backend   Redis/Gateway   Jenkins
                         │
                         ▼
                   observability/
                  Prometheus/Grafana
                  Loki/Alloy/Alertmanager
```

> 전체 논리/물리 아키텍처는 `seokpan-docs`에서 관리합니다.

---

## Repository Responsibility

`seokpan-gitops`는 **Kubernetes에서 어떤 상태를 유지할 것인가(Desired State)**를 관리합니다.

### Application

`apps/`

Frontend와 Backend 등 실제 서비스 Application의 Kubernetes Resource를 관리합니다.

```text
apps/
├── backend/
└── frontend/
```

주요 대상:

* Deployment
* Service
* ConfigMap
* HPA
* NetworkPolicy
* Kustomize Resource

Application의 실제 소스코드는 `seokpan-app`에서 관리합니다.

---

### Platform

`platform/`

Application이 실행되기 위해 필요한 공통 Runtime Platform을 관리합니다.

주요 대상:

* Redis
* Gateway
* NFS Storage Client
* Namespace / RBAC

Platform 구성은 Application 자체와 분리하여 관리합니다.

---

### CI/CD

`cicd/`

Application Build와 Delivery에 필요한 CI/CD Runtime을 관리합니다.

주요 대상:

* Jenkins Controller
* Jenkins JCasC
* Jenkins Plugin
* BuildKit Agent
* BuildKit Configuration
* CI/CD 관련 RBAC / PVC / Service

Application Source의 Build → Image → GitOps 변경 흐름은 다음과 같이 연결됩니다.

```text
seokpan-app
    │
    ▼
Jenkins
    │
    ▼
BuildKit
    │
    ▼
Harbor
    │
    ▼
Image Digest
    │
    ▼
seokpan-gitops
    │
    ▼
Argo CD
    │
    ▼
Kubernetes
```

---

### Observability

`observability/`

Cluster 및 Application 상태를 관측하기 위한 Runtime을 관리합니다.

주요 대상:

* Prometheus
* Grafana
* Loki
* Grafana Alloy
* Alertmanager
* External Exporter
* Monitoring 관련 Service / ConfigMap / NetworkPolicy

---

## Argo CD Structure

Child Application 선언은 `argocd/applications/`에서 관리합니다.

```text
argocd/
└── applications/
    ├── gateway.yaml
    ├── jenkins.yaml
    ├── namespaces-rbac.yaml
    ├── observability.yaml
    ├── redis.yaml
    └── storage-nfs.yaml
```

전체적인 동작은 다음과 같습니다.

```text
Git Repository
      │
      ▼
argocd/applications/
      │
      ▼
Argo CD Root Application
      │
      ▼
Child Application
      │
      ▼
Desired State Path
      │
      ▼
Kubernetes Cluster
```

`argocd/applications/`는 **Child Application 선언 영역**이며 실제 Workload Manifest는 각 책임 영역의 Directory에서 관리합니다.

---

## Repository Structure

```text
seokpan-gitops/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   └── pull_request_template.md
│
├── argocd/
│   └── applications/
│       ├── gateway.yaml
│       ├── jenkins.yaml
│       ├── namespaces-rbac.yaml
│       ├── observability.yaml
│       ├── redis.yaml
│       └── storage-nfs.yaml
│
├── apps/
│   ├── README.md
│   ├── backend/
│   └── frontend/
│
├── platform/
│   ├── gateway/
│   ├── namespaces-rbac/
│   ├── redis/
│   └── storage-nfs/
│
├── cicd/
│   └── Jenkins / BuildKit 구성
│
├── observability/
│   └── Monitoring / Logging 구성
│
├── CONTRIBUTING.md
├── .gitignore
└── README.md
```

---

## Current State

현재 Repository에는 다음 영역이 실제 Desired State로 구성되어 있습니다.

| 영역                            | 상태                                                    |
| ----------------------------- | ----------------------------------------------------- |
| Argo CD Application 구조        | 구성 완료                                                 |
| Namespace / RBAC              | Argo CD 관리 편입                                         |
| Gateway                       | Desired State 및 Child Application 구성                  |
| Redis                         | Runtime Desired State 구성                              |
| NFS Storage                   | Provisioner / StorageClass 관리                         |
| Jenkins                       | Controller / JCasC / BuildKit 구성                      |
| Observability                 | Prometheus / Grafana / Loki / Alloy / Alertmanager 구성 |
| Backend                       | Desired State 기반 구성                                   |
| Frontend                      | Desired State 기반 구성                                   |
| Application Image 자동 Delivery | Jenkins / Harbor / GitOps 연계 진행                       |

Backend와 Frontend는 GitOps 구조가 준비되어 있지만, **Desired State가 존재한다는 것과 실제 Application Runtime 통합이 완료되었다는 것은 구분합니다.**

---

## Application Deployment Flow

Application 배포의 기본 흐름은 다음과 같습니다.

```text
Developer
   │
   ▼
seokpan-app
   │
   │ Commit
   ▼
Jenkins
   │
   ├── Test
   ├── Build
   ├── Image Push
   │
   ▼
Harbor
   │
   │ Image Digest
   ▼
GitOps Manifest Update
   │
   ▼
Pull Request
   │
   ▼
Merge
   │
   ▼
Argo CD
   │
   ▼
Kubernetes
```

GitOps Repository에서는 Cluster에 직접 `kubectl apply`하는 것을 기본 배포 방식으로 사용하지 않습니다.

**Git 변경 → PR → Merge → Argo CD Sync**를 기본 흐름으로 사용합니다.

---

## GitOps Responsibility Boundary

각 Repository의 책임은 다음과 같이 분리합니다.

| Repository       | Responsibility                                                         |
| ---------------- | ---------------------------------------------------------------------- |
| `seokpan-infra`  | Host / VM / Network / Kubernetes Bootstrap / Infrastructure Automation |
| `seokpan-gitops` | Kubernetes Desired State / Argo CD                                     |
| `seokpan-app`    | Frontend / Backend Application Source                                  |
| `seokpan-docs`   | Project-wide Design / Architecture / Decision / Troubleshooting        |

### 변경 책임 예시

```text
VM / Network 변경
→ seokpan-infra

Kubernetes Bootstrap 변경
→ seokpan-infra

Kubernetes Deployment / Service 변경
→ seokpan-gitops

Application Code 변경
→ seokpan-app

Application 기능 설계 변경
→ seokpan-docs + seokpan-app

Project Architecture 변경
→ seokpan-docs
```

---

## Runtime Ownership

동일한 Kubernetes Resource를 여러 시스템이 동시에 관리하지 않습니다.

```text
Infrastructure
    │
    └── seokpan-infra
          └── Kubernetes Bootstrap

Kubernetes Desired State
    │
    └── seokpan-gitops
          └── Argo CD

Application Source
    │
    └── seokpan-app
```

특히 Argo CD가 관리하는 Resource를 Cluster에서 직접 수정한 뒤 Git을 나중에 맞추는 방식을 기본 운영 방식으로 사용하지 않습니다.

---

## GitOps Change Flow

Repository 변경은 다음 흐름을 따릅니다.

```text
Issue
  │
  ▼
Branch
  │
  ▼
Manifest 변경
  │
  ▼
Static Validation
  │
  ▼
Pull Request
  │
  ▼
Review
  │
  ▼
Merge
  │
  ▼
Argo CD Sync
  │
  ▼
Runtime Validation
```

변경 성격에 따라 Merge 전 정적 검증과 Merge 후 Runtime 검증을 구분합니다.

---

## Validation

대표적인 사전 검증:

* YAML 문법 검증
* Kustomize Render
* Kubernetes API Server Server-side Dry Run
* Resource 간 참조 확인
* Image Tag / Digest 확인
* Secret / Credential 미포함 확인
* Argo CD Application Source Path 확인

Runtime 변경이 포함되는 경우:

* Argo CD `Synced / Healthy`
* Pod `Running / Ready`
* Service / Endpoint 정상
* PVC / PV 상태
* Application Health
* 관련 Network / Gateway 연결

을 변경 범위에 따라 확인합니다.

---

## Security

Repository에는 다음 민감정보를 저장하지 않습니다.

* Password
* Token
* Private Key
* kubeconfig Credential
* Database Credential
* 실제 Kubernetes Secret 값

공개 인증서와 같은 비밀이 아닌 구성값과 실제 Secret을 구분하여 관리합니다.

---

## Related Repositories

### Application

`seokpan-app`

Frontend / Backend Source Code와 Application Test를 관리합니다.

### Infrastructure

`seokpan-infra`

Host / VM / Network / Kubernetes Bootstrap 및 Infrastructure Automation을 관리합니다.

### Documentation

`seokpan-docs`

프로젝트 요구사항, Logical / Physical Architecture, 설계 결정, 변경 이력 및 Troubleshooting을 관리합니다.

---

## Detailed Documentation

Repository 내부의 세부 구성은 각 Directory README를 우선 참고합니다.

```text
apps/README.md
apps/backend/README.md
apps/frontend/README.md
cicd/README.md
observability/README.md
argocd/applications/README.md
CONTRIBUTING.md
```

Project-wide 설계 및 Architecture는 `seokpan-docs`를 참고합니다.

---

## Collaboration

기본 변경 절차:

```text
Issue
→ Branch
→ Commit
→ Pull Request
→ Review
→ Squash Merge
→ Argo CD Sync
→ Runtime Validation
```

GitOps 변경은 **Cluster 상태를 직접 변경하는 것이 아니라 Git을 통해 Desired State를 변경하는 것**을 기본 원칙으로 합니다.
