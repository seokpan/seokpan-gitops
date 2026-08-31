# observability/

kube-prometheus-stack 값 오버라이드, ServiceMonitor, Alertmanager Receiver,
Grafana Dashboard Provisioning 등 관측성 스택 매니페스트.

M-01~M-05 검증 축(동시성/투표 처리 성능/장애 복구/DR/Ansible 개선 효과) 대시보드 및
알림 규칙이 이 경로에 위치합니다.

담당: 최유준 (Delivery & Observability)

## 저장소 방식 (정정: 2026-08-28)

Prometheus/Loki는 NFS 동적 PVC가 아니라 **정적 Local PV(hostPath)** 를 사용합니다.
(06 Ansible 자동화·테스트 설계, 11 Runbook 기준)

| 대상 | 방식 | 보존 범위 |
|---|---|---|
| Prometheus | 정적 Local PV (hostPath) | 7일 또는 20GiB, Node 종속 — 유실 수용 |
| Loki | Single Binary + 정적 Local PV (hostPath) | 72시간 또는 10GiB, Node 종속 — 유실 수용 |
| Alertmanager | 기본 PVC 없음 | — |
| Grafana | Provisioning 파일 기반 복원 (PVC 불필요) | UI 직접 변경 아님 |


## Root Application Sync 전 선행 조건 (정정판, 08/28)

`apps/root`(Root Application)에 이 디렉토리를 하위 Application으로 추가하기 전에
아래가 먼저 준비되어 있어야 합니다.

| # | 항목 | 이유 | 필요 시점 |
|---|---|---|---|
| 1 | Prometheus/Loki 배치 Node 및 hostPath 확정 → Inventory/host_vars/version-lock.yml에 고정 | 정적 Local PV는 특정 Node에 종속되므로 사전에 Node를 정하고 PV manifest에 nodeAffinity로 못박아야 함 (예시: worker-01, /mnt/observability/prometheus) | observability Application을 Root에 추가하기 전까지 |
| 2 | 관측성용 NetworkPolicy 사전 허용 규칙 | Calico가 단계적 Default Deny로 전환 중 — Prometheus→kube-state-metrics/kubelet, Grafana→Prometheus/Loki 통신이 배포 직후 막히지 않도록 선행 필요 | observability Sync 시점 전까지 |
| 3 | CoreDNS / hosts block 확인 | grafana.seokpan.soldesk.store 등 서비스명 해석 및 UI 노출(Gateway 연결) 전에 hosts 매핑 반영 여부 확인 필요 | Grafana/Prometheus UI 노출(Gateway 연결) 전까지 |

(참고) NFS Subdir Provisioner 자체는 클러스터에 여전히 필요합니다 — Redis/Jenkins Controller/MariaDB Backup
Staging이 사용하기 때문입니다. 다만 observability Application의 Sync를 막는 선행 조건은 아닙니다.

이 항목들은 이유빈 님(네트워크/hosts) 및 정태훈 님(Worker Node 배치)과 겹치므로,
observability 매니페스트를 Root에 연결하기 전 팀과 상태를 한 번 더 확인합니다.

---

## Application ServiceMonitor 보류 (신규, 2026-08-31)

`servicemonitor-app.yaml`은 Backend(FastAPI/Redis) Service 매니페스트가 `apps/` 하위에
아직 존재하지 않아 `servicemonitor-app.yaml.pending`으로 확장자를 바꿔 Root Application
동기화 대상에서 제외했습니다(`kubectl apply -f`/Argo CD 디렉토리 소스 모두 `.yaml`만 인식).

### 확정된 규칙 (팀 합의, 2026-08-31)

| 항목 | 값 |
|---|---|
| Namespace | `application` |
| Service Port 이름 | `http` |
| Metrics 경로 | `/metrics` |

### 재활성화 조건

1. `apps/` 하위에 Backend Service 매니페스트가 실제로 배포됨
2. 정태훈 님과 실제 Service Label이 위 표와 일치하는지 확인
   (파일 내 `app.kubernetes.io/part-of: seokpan` selector는 가정값이며 실제 라벨로 교체 필요)
3. 위 확인 후 `.pending` 확장자를 제거(`servicemonitor-app.yaml`로 rename)하고 PR
작성자: 최유준
작성 날짜: 2026-08-28
