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

## Application ServiceMonitor

Application Metrics 수집은 GitOps #91을 기준으로 활성화합니다.

Backend Runtime은 현재 2 Replica로 운영 중이며 새 Metrics Image에서 각 Pod의 `/metrics` HTTP 200과 다음 Metric 노출을 실제 확인했습니다.

- `python_info`
- `process_cpu_seconds_total`
- `seokpan_http_requests_total`
- `seokpan_http_request_duration_seconds`

ServiceMonitor 선택 계약은 다음과 같습니다.

| 항목 | 값 |
|---|---|
| Target Namespace | `application` |
| Backend Service | `backend` |
| Service metadata label | `app.kubernetes.io/name: backend` |
| Service Port | `http:8000` |
| Metrics Path | `/metrics` |
| Scrape Interval | `15s` |
| ServiceMonitor Namespace | `observability` |
| Prometheus 선택 label | `release: kube-prometheus-stack` |

`ServiceMonitor.spec.selector.matchLabels`는 Service의 `metadata.labels`를 선택하며, Service의 `spec.selector`가 Pod를 선택하는 것과 구분합니다.

Application ServiceMonitor는 본 변경 Merge 이후 `observability/servicemonitor-app.yaml`을 통해 Git Desired State에서 관리합니다. Application Metrics 수집 완료는 Argo CD Sync 후 Prometheus Target `UP`과 실제 Application Metric Query를 확인한 뒤 판정합니다.

Observability Platform이 정상 운영 중이라는 사실만으로 Application Metrics 수집 완료를 의미하지 않습니다.

`/metrics`는 Cluster 내부 수집 대상으로 유지하며 Gateway public route에는 추가하지 않습니다.

관련 추적: GitOps #91, seokpan-app #78 / PR #80.
