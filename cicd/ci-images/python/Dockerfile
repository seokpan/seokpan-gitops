# 작성자: 최유준
# 작성 날짜: 2026-09-08
#
# CI 전용 Python 실행 이미지. seokpan-app PR #48에서 확정된 backend/Dockerfile의
# Python/uv 버전·digest를 그대로 재사용하여 CI 환경과 Runtime 이미지 간
# 버전 계약을 일치시킵니다. 이 파일을 수정하면 반드시 아래 digest도
# PR #48의 backend/Dockerfile 최신 상태와 다시 맞춰야 합니다.
FROM python:3.13.15-slim-trixie@sha256:9d2e5553305c7c7b0097999bb17187c69b921ccd6bc9d40e4bb5ebe652c00285
COPY --from=ghcr.io/astral-sh/uv@sha256:e85be844203885286c60ffad8a858d48afb6c5a5c237ca0e67f12e74b8f174b1 /uv /uvx /usr/local/bin/
