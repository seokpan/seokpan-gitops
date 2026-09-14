# observability/

kube-prometheus-stack 값 오버라이드, ServiceMonitor, Alertmanager Receiver,
Grafana Dashboard Provisioning 등 관측성 스택 매니페스트를 관리합니다.

M-01~M-05 검증 축(동시성/투표 처리 성능/장애 복구/DR/Ansible 개선 효과) 대시보드 및
알림 규칙이 이 경로에 위치합니다.

담당: 최유준 (Delivery & Observability)

## 저장소 방식

Prometheus/Loki는 NFS 동적 PVC가 아니라 **정적 Local PV(hostPath)** 를 사용합니다.

| 대상 | 방식 | 보존 범위 |
|---|---|---|
| Prometheus | 정적 Local PV (hostPath) | 7일 또는 20GiB, Node 종속 — 유실 수용 |
| Loki | Single Binary + 정적 Local PV (hostPath) | 72시간 또는 10GiB, Node 종속 — 유실 수용 |
| Alertmanager | 기본 PVC 없음 | — |
| Grafana | Provisioning 파일 기반 복원 (PVC 불필요) | UI 직접 변경 아님 |

NFS Subdir Provisioner는 Redis/Jenkins Controller/MariaDB Backup Staging 등 다른 영역에서 계속 사용하지만 Observability 저장소 방식과는 구분합니다.

## Argo CD 관리 상태

현재 Observability Child Application 선언은 `argocd/applications/observability.yaml`에서 관리합니다.

```text
Root Application
→ argocd/applications/observability.yaml
→ observability/
→ Kubernetes namespace: observability
```

`observability` Application은 `targetRevision: main`을 사용하고 automated `prune: true`, `selfHeal: true`, `ServerSideApply=true` 기준으로 Desired State를 관리합니다.

따라서 현재 운영 변경은 기존 `apps/root` 경로가 아니라 `argocd/applications/`와 `observability/`의 Git Desired State를 기준으로 수행합니다.

Prometheus/Loki의 Node·hostPath, Observability NetworkPolicy, CoreDNS/hosts 등 기존 선행조건은 신규 배포·재구축·회귀검증 시 계속 확인해야 하지만, 이미 Root에 편입된 Application을 아직 연결 전인 것처럼 표현하지 않습니다.

---

## Application ServiceMonitor 보류

Application용 ServiceMonitor 자산은 현재 `servicemonitor-app.yaml.pending`으로 유지하며 Argo CD 적용 대상에서 제외합니다.

Backend Service는 현재 `apps/backend/service.yaml`에 존재하고 `apps-backend` Child Application을 통해 GitOps 관리 대상에 편입되어 있습니다. 다만 Backend Runtime 자체는 아직 `replicas: 0`, `git-pending`이며 Application Metrics 수집도 검증되지 않았습니다.

### 현재 확정 계약

| 항목 | 값 |
|---|---|
| Namespace | `application` |
| Backend Service | `backend` |
| Service selector | `app.kubernetes.io/name: backend` |
| Service Port 이름 | `http` |
| Metrics 경로 | `/metrics` |

### 재활성화 조건

1. Backend Runtime이 실제로 활성화됨
2. Backend가 `/metrics`를 실제로 제공함
3. `servicemonitor-app.yaml.pending`의 selector가 실제 Backend Service label과 일치함
4. Service Port `http`와 ServiceMonitor endpoint가 일치함
5. 변경 내용을 PR로 검토한 뒤 `.pending`을 제거하여 정식 Desired State에 편입함
6. Merge/Sync 후 Prometheus Target과 Application Metrics 수집 상태를 확인함

`observability` Platform이 Running이라는 사실만으로 Application ServiceMonitor 또는 Application Metrics 수집 완료를 선언하지 않습니다.

관련 추적: GitOps #86, Docs #107.
