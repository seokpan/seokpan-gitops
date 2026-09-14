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

Backend Service는 `apps/backend/service.yaml`에 존재하고 `apps-backend` Child Application을 통해 GitOps 관리 대상에 편입되어 있습니다. 다만 Backend Runtime 자체는 아직 `replicas: 0`, `git-pending`이며 Application Metrics 수집도 검증되지 않았습니다.

또한 `ServiceMonitor.spec.selector.matchLabels`는 **Service의 `metadata.labels`** 를 선택합니다. 현재 Backend Service에는 ServiceMonitor가 선택할 `metadata.labels` 계약이 아직 정의되어 있지 않습니다. `Service.spec.selector`의 Pod 선택 라벨과 ServiceMonitor의 Service 선택 라벨을 같은 것으로 취급하지 않습니다.

### 현재 확인된 계약

| 항목 | 값 | 상태 |
|---|---|---|
| Namespace | `application` | 확정 |
| Backend Service | `backend` | 존재 |
| Service Port 이름 | `http` | 확정 |
| Metrics 경로 | `/metrics` | Runtime 미검증 |
| ServiceMonitor selector 후보 | `app.kubernetes.io/name: backend` | **후속 Service `metadata.labels` 계약 필요** |

### 이번 현행화 범위

- Root/Child Application 경로를 현재 구조로 정정
- Backend Service가 이미 존재한다는 현재 상태 반영
- ServiceMonitor가 아직 `.pending`인 이유를 Runtime 미활성·Metrics 미검증·Service metadata label 미확정으로 정리
- pending 파일의 selector를 **후속 metadata label 계약 후보값**으로만 명시

### 실제 재활성화 조건

실제 ServiceMonitor 활성화는 GitOps #91에서 별도 추적합니다.

1. Backend Runtime이 실제로 활성화됨
2. Backend가 `/metrics`를 실제로 제공함
3. Backend Service에 ServiceMonitor가 선택할 `metadata.labels` 계약을 결정·추가함
4. `servicemonitor-app.yaml.pending` selector와 Service `metadata.labels`가 실제로 일치함
5. Service Port `http`와 ServiceMonitor endpoint가 일치함
6. `.pending` 제거 전 server-side dry-run 및 Review 수행
7. Merge/Argo CD Sync 후 Prometheus Target과 Application Metrics 수집 상태 확인

`observability` Platform이 Running이라는 사실만으로 Application ServiceMonitor 또는 Application Metrics 수집 완료를 선언하지 않습니다.

관련 추적: GitOps #86, #91, seokpan-docs 11 Kubernetes & Application Integration Runbook.
