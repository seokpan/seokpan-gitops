# seokpan-gitops

**SeokPan 프로젝트의 Kubernetes 배포 상태와 Argo CD 설정을 관리하는 저장소입니다.**

이 저장소에서는 Kubernetes에 어떤 애플리케이션과 서비스를 **어떤 설정으로 실행할 것인지**를 파일로 관리합니다.

Argo CD는 이 저장소의 내용을 기준으로 Kubernetes 클러스터의 실제 상태를 계속 확인하고, 저장소에 정의된 상태와 다르면 Kubernetes를 다시 원하는 상태로 맞춥니다.

> 쉽게 말하면,
> **`seokpan-gitops`는 Kubernetes가 어떤 모습으로 실행되어야 하는지를 기록해 두는 저장소입니다.**

---

## 📑 목차

1. [프로젝트 소개](#-프로젝트-소개)
2. [전체 GitOps 구성](#-전체-gitops-구성)
3. [배포 흐름](#-배포-흐름)
4. [관리하는 영역](#-관리하는-영역)
5. [저장소 구성](#-저장소-구성)
6. [다른 저장소와의 관계](#-다른-저장소와의-관계)
7. [관련 문서](#-관련-문서)
8. [보안 관리](#-보안-관리)
9. [작업 방법](#-작업-방법)

---

## 📌 프로젝트 소개

SeokPan 프로젝트는 실시간 투표형 웹게임 오목 서비스를 Kubernetes 환경에서 실행합니다.

`seokpan-gitops`는 이 Kubernetes 환경에서 **애플리케이션과 운영 서비스를 어떤 상태로 실행할 것인지**를 관리합니다.

주요 관리 대상은 다음과 같습니다.

| 구분            | 관리 내용                                                |
| ------------- | ---------------------------------------------------- |
| Argo CD       | Kubernetes 배포를 관리하는 도구와 Application 설정               |
| Application   | Frontend / Backend 등 실제 서비스                          |
| Platform      | Gateway, Redis, Storage, Namespace, RBAC 등 공통 환경     |
| CI/CD         | Jenkins 등 빌드 및 배포 관련 서비스                             |
| Observability | Prometheus, Grafana, Loki, Alloy, Alertmanager 등     |
| Kubernetes 설정 | Service, Deployment, ConfigMap, HPA, NetworkPolicy 등 |

---

## 🏗️ 전체 GitOps 구성

SeokPan 프로젝트의 배포 구조는 다음과 같습니다.

```text
개발자가 코드 수정
       ↓
  seokpan-app
       ↓
     Jenkins
       ↓
Container Image Build
       ↓
     Harbor
       ↓
seokpan-gitops
       ↓
     Argo CD
       ↓
   Kubernetes
       ↓
새로운 Pod 실행
```

`seokpan-app`은 애플리케이션 소스코드를 관리하고, `seokpan-gitops`는 Kubernetes에 배포할 상태를 관리합니다.

Jenkins가 새로운 컨테이너 이미지를 만들고 Harbor에 저장한 이후, GitOps 저장소의 이미지 정보를 변경하면 Argo CD가 이를 확인하여 Kubernetes에 새로운 버전을 배포합니다.

### GitOps의 핵심 구조

```text
┌──────────────────────┐
│   seokpan-gitops     │
│                      │
│ Kubernetes Desired   │
│ State 관리           │
└──────────┬───────────┘
           │
           │ Argo CD Sync
           ↓
┌──────────────────────┐
│     Kubernetes       │
│                      │
│ 실제 실행 상태        │
└──────────────────────┘
```

여기서 **Desired State**는 “Kubernetes가 이렇게 실행되고 있어야 한다”라고 파일에 적어 놓은 원하는 상태를 의미합니다.

---

## 🔄 배포 흐름

애플리케이션 배포는 다음과 같은 흐름으로 동작합니다.

```text
① 개발자
   │
   │ 코드 변경
   ↓
② seokpan-app
   │
   │ CI 실행
   ↓
③ Jenkins
   │
   │ 이미지 생성
   ↓
④ Harbor
   │
   │ 이미지 저장
   ↓
⑤ seokpan-gitops
   │
   │ 이미지 Tag / Digest 변경
   ↓
⑥ Argo CD
   │
   │ Git 변경 감지
   ↓
⑦ Kubernetes
   │
   │ 새로운 이미지 Pull
   ↓
⑧ 새로운 Pod 실행
```

### GitOps에서 중요한 점

Kubernetes에 직접 접속해서 Deployment를 수정하는 대신, **Git 저장소의 파일을 수정하는 것을 기본적인 변경 방법**으로 사용합니다.

```text
잘못된 방법

개발자
  ↓
kubectl edit / kubectl apply
  ↓
Kubernetes
```

보다는 다음 방식을 사용합니다.

```text
권장 방식

개발자
  ↓
Git 변경
  ↓
Pull Request
  ↓
Merge
  ↓
Argo CD
  ↓
Kubernetes
```

이렇게 하면 Kubernetes에 어떤 변경이 있었는지 Git 기록을 통해 확인할 수 있습니다.

---

## 🧩 관리하는 영역

`seokpan-gitops`는 기능별로 Kubernetes 설정을 나누어 관리합니다.

### Argo CD

```text
argocd/applications/
```

Argo CD가 어떤 Application을 관리할지 정의합니다.

현재 구조에서는 Root Application이 여러 Child Application을 관리하는 **App-of-Apps 방식**을 사용합니다.

각 Application은 실제 Kubernetes 리소스를 직접 모두 포함하기보다 `apps`, `platform`, `cicd`, `observability`에 있는 설정을 연결합니다.

---

### 애플리케이션

```text
apps/
```

실제 서비스를 실행하기 위한 Kubernetes 설정을 관리합니다.

주요 대상:

* Frontend
* Backend
* Service
* HPA
* NetworkPolicy
* 애플리케이션에 필요한 ConfigMap 등

애플리케이션의 **소스코드 자체는 이 저장소에서 관리하지 않습니다.**

소스코드는 `seokpan-app`에서 관리하고, 이 저장소에서는 Kubernetes에서 실행할 방법을 관리합니다.

---

### 플랫폼

```text
platform/
```

애플리케이션이 실행되기 위해 필요한 공통 Kubernetes 환경을 관리합니다.

주요 대상:

* Gateway
* Redis
* Storage
* Namespace
* RBAC
* 공통 Kubernetes 설정

애플리케이션 자체와 관계없이 여러 서비스에서 함께 사용하는 환경을 이 영역에서 관리합니다.

---

### CI/CD

```text
cicd/
```

애플리케이션을 빌드하고 배포하는 데 필요한 Kubernetes 환경을 관리합니다.

주요 대상:

* Jenkins
* CI/CD 관련 설정

---

### 모니터링

```text
observability/
```

서비스가 정상적으로 동작하고 있는지 확인하기 위한 Kubernetes 환경을 관리합니다.

주요 대상:

* Prometheus
* Grafana
* Loki
* Grafana Alloy
* Alertmanager

로그, 메트릭, 알림 등을 이용하여 서비스와 Kubernetes 상태를 확인할 수 있도록 구성합니다.

---

## 📁 저장소 구성

현재 주요 디렉터리는 다음과 같습니다.

```text
seokpan-gitops/
├── argocd/
│   └── applications/
│
├── apps/
│
├── platform/
│
├── cicd/
│
├── observability/
│
├── .gitignore
└── README.md
```

### 디렉터리 역할

| 경로                     | 설명                          |
| ---------------------- | --------------------------- |
| `argocd/applications/` | Argo CD Application 관리      |
| `apps/`                | Frontend / Backend 등 실제 서비스 |
| `platform/`            | Kubernetes 공통 환경            |
| `cicd/`                | Jenkins 등 CI/CD 환경          |
| `observability/`       | 모니터링 및 로그 환경                |
| `.gitignore`           | Git에 저장하지 않을 파일 관리          |
| `README.md`            | 저장소 설명                      |

---

## 🔗 다른 저장소와의 관계

SeokPan 프로젝트는 각 저장소의 역할을 나누어서 관리합니다.

| 저장소              | 담당 내용                        |
| ---------------- | ---------------------------- |
| `seokpan-app`    | 애플리케이션 소스코드                  |
| `seokpan-infra`  | 서버 / 네트워크 / 데이터베이스 / 인프라 자동화 |
| `seokpan-gitops` | Kubernetes 배포 상태 / Argo CD   |
| `seokpan-docs`   | 설계 / 테스트 / 검증 / 변경 기록        |

전체적인 관계를 보면 다음과 같습니다.

```text
                    ┌─────────────────┐
                    │  seokpan-app    │
                    │ 애플리케이션 코드 │
                    └────────┬────────┘
                             │
                             ↓
                         Jenkins
                             │
                             ↓
                          Harbor
                             │
                             │ Image
                             ↓
                    ┌─────────────────┐
                    │ seokpan-gitops  │
                    │ Kubernetes 상태  │
                    └────────┬────────┘
                             │
                             ↓
                         Argo CD
                             │
                             ↓
                    ┌─────────────────┐
                    │   Kubernetes    │
                    │ 실제 서비스 실행  │
                    └─────────────────┘


┌─────────────────┐
│ seokpan-infra   │
│ 인프라 구성/자동화 │
└─────────────────┘

┌─────────────────┐
│  seokpan-docs   │
│ 설계/검증/문서화  │
└─────────────────┘
```

### 저장소별 간단한 구분

**`seokpan-app`**

> 무엇을 실행할 것인가?

애플리케이션의 실제 소스코드를 관리합니다.

**`seokpan-infra`**

> 어디에서 실행할 수 있도록 만들 것인가?

서버, 네트워크, 데이터베이스, Kubernetes 기반 환경 등을 구성합니다.

**`seokpan-gitops`**

> Kubernetes에서 어떻게 실행할 것인가?

Deployment, Service, HPA 등의 Kubernetes Desired State를 관리합니다.

**`seokpan-docs`**

> 왜 이렇게 만들었고, 어떻게 확인했는가?

프로젝트 설계, 테스트, 검증 및 변경 내용을 관리합니다.

---

## 📚 관련 문서

GitOps의 상세한 설계와 전체 아키텍처는 `seokpan-docs`에서 관리합니다.

README에서는 전체 내용을 반복해서 작성하지 않고 필요한 상세 문서로 이동할 수 있도록 구성합니다.

### 아키텍처

* 전체 논리 아키텍처
* 전체 물리 아키텍처
* Kubernetes 및 서비스 구성

### GitOps

* Kubernetes 배포 구조
* Argo CD 구성
* 애플리케이션 배포 흐름

### 프로젝트 기준

* MVP 구현 기준
* 프로젝트 변경 기록
* 테스트 및 검증 문서

> `seokpan-gitops`에서는 **실제 Kubernetes 설정을 관리**하고,
> `seokpan-docs`에서는 **그 설정을 왜 사용하고 어떻게 검증했는지**를 관리합니다.

---

## 🔐 보안 관리

GitOps 저장소는 Kubernetes에서 사용할 설정을 관리하지만, 실제 비밀번호나 개인 인증 정보를 Git에 저장하지 않습니다.

다음과 같은 민감한 정보는 저장소에 그대로 저장하지 않습니다.

* Password
* Token
* Private Key
* kubeconfig 인증 정보
* 실제 Kubernetes Secret 값
* 기타 개인 인증 정보

특히 Kubernetes Secret을 YAML 파일에 작성할 때도 **실제 비밀번호나 Token을 그대로 Commit하지 않는 것**을 기본 원칙으로 합니다.

---

## 🔄 작업 방법

Kubernetes 설정을 변경할 때는 다음과 같은 흐름을 사용합니다.

```text
Issue 등록
   ↓
작업 브랜치 생성
   ↓
Kubernetes 설정 수정
   ↓
로컬 / 테스트 환경 검증
   ↓
Commit
   ↓
Pull Request
   ↓
코드 리뷰
   ↓
Merge
   ↓
Argo CD Sync
   ↓
Kubernetes 반영
```

### 기본 원칙

**1. Git을 통해 변경합니다.**

Kubernetes에 직접 접속하여 설정을 변경하기보다 Git 저장소의 파일을 변경합니다.

**2. Pull Request를 사용합니다.**

변경 내용을 다른 팀원이 확인할 수 있도록 PR을 생성합니다.

**3. Merge된 내용을 기준으로 배포합니다.**

검토가 완료된 변경 내용만 `main`에 반영합니다.

**4. Argo CD가 Kubernetes에 반영합니다.**

Git의 Desired State와 Kubernetes의 실제 상태를 비교하고 필요한 변경을 적용합니다.

---

## 🎯 정리

`seokpan-gitops`는 SeokPan 프로젝트의 **Kubernetes 배포 상태를 코드로 관리하는 저장소**입니다.

전체 흐름은 다음과 같습니다.

```text
애플리케이션 코드
      ↓
    Jenkins
      ↓
    Harbor
      ↓
seokpan-gitops
      ↓
    Argo CD
      ↓
 Kubernetes
```

즉,

> **`seokpan-app`은 애플리케이션을 관리하고,
> `seokpan-infra`는 실행 환경을 만들고,
> `seokpan-gitops`는 Kubernetes에서 실행할 상태를 관리하며,
> `seokpan-docs`는 전체 설계와 검증 내용을 관리합니다.**

이렇게 저장소별 역할을 나누어 관리함으로써 **코드 변경 → 이미지 생성 → 배포 상태 변경 → Kubernetes 배포**까지의 과정을 Git 기록으로 확인할 수 있도록 구성합니다.
