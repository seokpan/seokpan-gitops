# cicd/

Jenkins Controller Deployment/PVC/Service, JCasC(Jenkins Configuration as Code) ConfigMap 등
CI/CD 파이프라인 인프라 매니페스트.

담당: 최유준 (Delivery & Observability)

## 이미지 태그 규칙 (확정: git-<commit-sha>)

06 설계서(기술 비교 및 논리 아키텍처 문서) 8단계 CI/CD 파이프라인 기준으로
`git-` prefix를 포함한 `git-<commit-sha>` 형식으로 확정합니다. (정태훈 님 질의에 대한 회신, 08/28)

- `harbor.seokpan.soldesk.store/seokpan/frontend:git-<sha>`
- `harbor.seokpan.soldesk.store/seokpan/backend:git-<sha>`

Harbor는 동일 Tag 재푸시를 거부하는 정책이 걸려 있는데, 커밋 단위로 유일성이 보장되는
`git-<sha>` 형식이 이 정책과도 자연스럽게 맞습니다(순수 SHA만 쓰는 것과 실질적 차이는 prefix 유무뿐).

## Rootless BuildKit 인증 방식

- `docker login`을 사용하지 않는 구조.
- `~/.docker/config.json` 형식의 credential 파일을 Kubernetes Secret으로 생성 → Jenkins Agent Pod에 마운트,
  `DOCKER_CONFIG` 환경변수로 참조.

## Harbor 접근 권한 (확정, 08/28)

- Admin Credential 사용 금지.
- Push/Pull 범위만, `seokpan` Project 한정 Robot Account 발급 → **harbor role 자동화 범위**
  (11 Runbook: "Harbor 설치·Project·Robot Account·GC | Ansible(harbor Role)") — 별도 role이 아니라
  기존 harbor role에 Task 추가.
- Vault 저장: `group_vars/vault.yml` 단일 파일 (06 문서 기준, 08/28 확정 — 기존 host_vars/harbor/vault.yml
  관행에서 전환. 기존 항목 마이그레이션은 별도 작업으로 남음).
- **Secret 경계 원칙 (06 문서 4.1)**: Argo CD/GitOps repo는 Secret 참조만 관리하고 실제 값을 갖지 않음.
  Robot Account 실제 값은 Ansible이 클러스터에 K8s Secret으로 직접 주입(`jenkins-harbor-credential`,
  `harbor-robot-dockerconfig`) — seokpan-gitops repo에는 절대 커밋하지 않음.
  JCasC ConfigMap은 `${HARBOR_ROBOT_USERNAME}` 같은 환경변수 치환으로만 참조.

## Jenkinsfile 작성 시 주의사항

- Push 후 Harbor UI/API에서 Digest 등록 확인 단계 포함.
- 동일 Tag 재푸시는 거부되므로, 실패 시 재시도 로직이 기존 태그를 그대로 재사용하지 않도록 설계
  (재시도 시 신규 커밋/빌드 번호를 태그에 반영하거나, 실패한 태그는 Harbor에서 정리 후 재시도).

---
작성자: 최유준
작성 날짜: 2026-08-28
