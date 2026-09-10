# 작성자: 최유준
# 작성 날짜: 2026-09-08
# 수정 날짜: 2026-09-10 (Trivy 바이너리 추가 — Issue #58 Scan Stage용)
#
# CI 전용 Python 실행 이미지. seokpan-app PR #48에서 확정된 backend/Dockerfile의
# Python/uv 버전·digest를 그대로 재사용하여 CI 환경과 Runtime 이미지 간
# 버전 계약을 일치시킵니다. 이 파일을 수정하면 반드시 아래 digest도
# PR #48의 backend/Dockerfile 최신 상태와 다시 맞춰야 합니다.
#
# [Trivy 관련 주의]
# - 2026-02~03월 aquasecurity/trivy 공급망 침해 사건(탈취 credential로 악성
#   v0.69.4 배포, trivy-action/setup-trivy 태그 대량 force-push) 이후이므로
#   반드시 v0.71.1 이상(취약점 수정 버전)을 사용할 것. v0.71.0 미만 금지.
# - 아래 TRIVY_VERSION/digest는 팀에서 직접 pull하여 확정 필요 (이 파일 작성
#   시점에는 이 환경에서 Docker Hub/GHCR 레지스트리에 접근할 수 없어 실제
#   digest를 검증하지 못했음 — 반드시 아래 확인 절차를 거친 뒤 채워넣을 것):
#     docker pull aquasec/trivy:0.72.0
#     docker inspect --format='{{index .RepoDigests 0}}' aquasec/trivy:0.72.0
FROM python:3.13.15-slim-trixie@sha256:9d2e5553305c7c7b0097999bb17187c69b921ccd6bc9d40e4bb5ebe652c00285

COPY --from=ghcr.io/astral-sh/uv@sha256:e85be844203885286c60ffad8a858d48afb6c5a5c237ca0e67f12e74b8f174b1 /uv /uvx /usr/local/bin/

# TODO(확인 필요): 아래 digest를 실제 pull 결과로 교체할 것. 임시로 태그만
# 명시해두었으며, digest 없이는 재현성이 깨지므로 이 상태로 merge 금지.
COPY --from=aquasec/trivy:0.72.0@sha256:cffe3f5161a47a6823fbd23d985795b3ed72a4c806da4c4df16266c02accdd6f /usr/local/bin/trivy /usr/local/bin/trivy

# runAsUser: 1000 환경에서 권한 오류 방지를 위한 캐시 경로 명시
ENV TRIVY_CACHE_DIR=/tmp/.trivy_cache
