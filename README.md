# seokpan-gitops

**「石나가는 판단(Seokpan)」의 Kubernetes 배포 상태(Desired State)와 Argo CD 설정을 관리하는 저장소입니다.** Kubernetes에 무엇을 어떤 설정으로 실행할지 파일로 정의하고, Argo CD가 이 저장소 기준으로 클러스터 실제 상태를 지속적으로 맞춥니다.

애플리케이션 소스는 [`seokpan-app`](https://github.com/seokpan/seokpan-app), 서버/네트워크/K8s 부트스트랩은 [`seokpan-infra`](https://github.com/seokpan/seokpan-infra), 설계·검증 문서는 [`seokpan-docs`](https://github.com/seokpan/seokpan-docs)에서 관리합니다.

## 배포 흐름

```text
개발자 코드 변경 → seokpan-app → Jenkins(빌드) → Harbor(이미지 저장)
→ seokpan-gitops(이미지 Tag/Digest 변경) → Argo CD(Git 변경 감지) → Kubernetes(새 Pod 실행)
```

Kubernetes를 `kubectl edit/apply`로 직접 바꾸지 않고, 이 저장소의 파일을 PR로 변경 → merge → Argo CD Sync로 반영하는 것이 기본 원칙입니다.

## 저장소 구조

```text
seokpan-gitops/
├── argocd/applications/   # App-of-Apps 방식 Argo CD Application 정의
├── apps/                  # 실제 서비스(Frontend/Backend) K8s 설정 — 소스코드 자체는 없음
│   ├── backend/
│   ├── frontend/
│   └── README.md
├── platform/              # Gateway, Redis, Storage, Namespace, RBAC 등 공통 환경
├── cicd/                  # Jenkins 등 CI/CD Kubernetes 환경
├── observability/         # Prometheus, Grafana, Loki, Alloy, Alertmanager
└── CONTRIBUTING.md
```

각 디렉터리의 세부 리소스 구성은 [`apps/README.md`](apps/README.md) 및 각 하위 폴더에서 직접 확인합니다.

## 관리 영역 요약

| 영역 | 경로 | 주요 대상 |
|---|---|---|
| Argo CD | [`argocd/applications/`](argocd/applications) | App-of-Apps 루트/자식 Application |
| 애플리케이션 | [`apps/`](apps) | Frontend/Backend Deployment, Service, ConfigMap |
| 플랫폼 공통 | [`platform/`](platform) | Gateway, Redis, Storage(NFS Provisioner), Namespace, RBAC |
| CI/CD | [`cicd/`](cicd) | Jenkins |
| Observability | [`observability/`](observability) | Prometheus, Grafana, Loki, Alloy, Alertmanager |

> HPA, NetworkPolicy 등은 아직 저장소에 없으며, 도입 시 `apps/`·`platform/`에 추가합니다.

## 작업 방법

Issue 등록 → 브랜치 생성 → 매니페스트 수정 → 로컬/테스트 검증 → PR → 리뷰 → `main` merge → Argo CD Sync 순으로 진행합니다. 상세 컨벤션은 [`CONTRIBUTING.md`](CONTRIBUTING.md) 참고. Kubernetes Secret에 실제 비밀번호/Token을 평문 커밋하지 않습니다.

## 저장소 간 관계

| 저장소 | 역할 |
|---|---|
| `seokpan-app` | 애플리케이션 소스코드 |
| `seokpan-infra` | 서버 / 네트워크 / DB / K8s 부트스트랩 자동화 |
| `seokpan-gitops` | Kubernetes Desired State / Argo CD (본 저장소) |
| `seokpan-docs` | 설계 / 검증 / 변경 기록 |

## 관련 문서 (`seokpan-docs`)

- [MVP 구현 기준](https://github.com/seokpan/seokpan-docs/blob/main/MVP_IMPLEMENTATION_BASELINE.md) · [프로젝트 변경 기록](https://github.com/seokpan/seokpan-docs/blob/main/PROJECT_CHANGES.md)
- [Kubernetes Application Integration 실시설계](<https://github.com/seokpan/seokpan-docs/tree/main/09_MVP_실행·통합_실시설계>)
- [Kubernetes Application Integration Runbook](<https://github.com/seokpan/seokpan-docs/tree/main/11_MVP_구축·자동화_Runbook>)
- [Kubernetes Application Integration 검증 계획](<https://github.com/seokpan/seokpan-docs/tree/main/12_MVP_검증·측정_계획>)
- [Redis Runtime Resource Baseline](https://github.com/seokpan/seokpan-docs/blob/main/REDIS_RUNTIME_RESOURCE_BASELINE.md)