# CONTRIBUTING — Branch / PR 정책

`seokpan-gitops` 저장소에 적용되는 협업 규칙입니다. `seokpan-infra`(Ansible) 저장소와
동일한 팀 컨벤션을 따르며, **main 브랜치 직접 push는 금지**합니다.

## 기본 흐름

```
Issue 생성 → Branch 생성 → Commit → PR → Review → Merge
```

1. **Issue** — 작업 시작 전 GitHub Issue로 등록 (무엇을/왜 하는지 1~2줄)
2. **Branch** — 아래 네이밍 규칙에 따라 Issue에서 분기
3. **Commit** — 의미 단위로 커밋, 커밋 메시지에 변경 이유 포함
4. **PR** — main 대상으로 PR 생성, 아래 PR 작성 규칙 따름
5. **Review** — 최소 1인 승인 필요, Argo CD Application/Root 관련 변경은 정태훈 님 리뷰 필수
6. **Merge** — 승인 후 Merge, Merge 즉시 Argo CD가 감지하여 Sync

## 브랜치 네이밍 규칙

```
<issue번호>_<도메인>/<짧은-설명>
```

예: `12_infra/root-application-temp`

- `도메인`: `infra`, `cicd`, `observability`, `platform`, `apps` 등 변경 대상 디렉토리와 일치시킵니다.
- `-temp` 접미사: 검증/실험 후 버릴 브랜치임을 표시. PR 없이 로컬에서 확인만 하고
  `git branch -D`로 삭제하는 용도이며, Merge 대상이 아닙니다.
- 실제 Merge까지 갈 정식 작업 브랜치는 `-temp` 없이 명명합니다.

## Commit 메시지

- 한 커밋 = 하나의 논리적 변경 단위 (매니페스트 추가 + 오타 수정 등을 한 커밋에 섞지 않음)
- 형식 예시: `[cicd] Jenkins Controller Deployment 매니페스트 추가`
- CRD/대형 리소스 변경처럼 이유가 명확하지 않은 커밋은 본문에 배경 1~2줄 추가

## PR 작성 규칙

- **본문은 항목당 1~2줄로 간결하게** — 무엇을 왜 바꿨는지만
- **리뷰어가 집중해서 봐야 할 파일을 명시적으로 지정**
  (예: "`cicd/jenkins-controller-deployment.yaml`의 PVC 크기만 확인해주세요")
- 관련 Issue 번호 링크 (`Closes #12`)
- Argo CD Sync에 영향 있는 변경(경로 추가/삭제, Application 매핑 변경)은 PR 상단에 별도로 표시

## Merge 이후 (Rollback 경로)

Sync 후 문제가 발견되면 `kubectl` 수동 수정이 아니라 **Git Revert → Merge → Argo CD Sync**
순서로 되돌립니다. (argocd 네임스페이스는 Self-Heal이 걸려 있어 수동 변경은 자동으로 되돌아갑니다.)

## GitHub 저장소 설정 (Branch Protection, 확정)

아래는 GitHub 저장소 설정으로 이미 강제되고 있는 규칙입니다. 이 문서는 해당 설정을 문서화한 것이며,
설정 자체를 바꾸려면 저장소 Settings에서 별도로 변경해야 합니다.

- **Merge 전략: Squash only** — PR의 커밋 여러 개가 main에는 1개로 합쳐집니다. 따라서 PR 제목이
  실질적으로 main의 커밋 메시지가 되므로, PR 제목을 명확하게 작성합니다.
- **필수 리뷰어: 1명** — 4인 중 누구든 1명 승인하면 Merge 가능합니다. Argo CD Application/Root
  관련 변경처럼 영향도가 큰 PR은 정태훈 님 리뷰를 우선 요청하는 걸 팀 관례로 두는 걸 권장합니다
  (강제 규칙은 아님).

---
작성자: 최유준
작성 날짜: 2026-08-28
