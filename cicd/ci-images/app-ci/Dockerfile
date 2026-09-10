# 작성자: 최유준
# 작성 날짜: 2026-09-09
#
# App #62 Jenkins 검사 연결용 통합 CI 이미지 (app-ci).
# Backend 설치·검사 -> OpenAPI -> Frontend 설치·검사 -> UI Browser -> Full Browser ->
# 최종 집계까지 하나의 컨테이너/파일시스템에서 실행하기 위한 이미지.
#
# [Base 선택 이유] python:3.13.15-slim-trixie(Debian trixie, glibc)를 그대로 재사용한다.
# ci-images/python/Dockerfile, seokpan-app/backend/Dockerfile과 동일 Base/Digest를 유지해야
# Backend 단계에서 만든 backend/.venv가 Full Browser 단계(같은 컨테이너)에서 그대로 spawn 가능하다
# (다른 libc 이미지로 만들면 venv의 Python 바이너리가 이 컨테이너에서 실행 실패할 수 있음).
#
# [Node 설치 방식] node:24.19.0의 Alpine(musl) 변형을 쓰지 않는다.
# frontend/Dockerfile은 musl 기반이지만 그건 서비스 배포 이미지일 뿐이고, 여기는
# Python(glibc)과 반드시 같은 libc여야 하므로 Node 공식 이미지의 Debian(trixie-slim) 변형에서
# /usr/local 트리만 복사해온다. 기존 uv 설치 방식(COPY --from=...)과 동일 패턴 재사용.
# Digest는 cp-03에서 kubectl run(sleep 3600) 후 imageID로 확보 (2026-09-09).
#
# [Playwright Chromium 위치 - 별도 확인 필요] frontend/scripts/browser-e2e.mjs가
# PLAYWRIGHT_BROWSERS_PATH를 frontend/.browser-cache로 코드에서 고정한다(App 소스 변경 금지).
# 이 경로는 Jenkins Workspace Mount에 덮여 사라지므로, 이 이미지는 Workspace 바깥 경로
# (/opt/playwright-cache)에 Chromium을 구워두기만 한다. 실제 frontend/.browser-cache로는
# Jenkinsfile의 별도 Stage에서 복사(symlink 아님 — 팀 확정, 이유: Python/Node 검사 도구가
# Checkout 안 심볼릭 링크를 거부함)한다.

FROM python:3.13.15-slim-trixie@sha256:9d2e5553305c7c7b0097999bb17187c69b921ccd6bc9d40e4bb5ebe652c00285

COPY --from=ghcr.io/astral-sh/uv@sha256:e85be844203885286c60ffad8a858d48afb6c5a5c237ca0e67f12e74b8f174b1 \
    /uv /uvx /usr/local/bin/

# Node 24.19.0 (Debian trixie-slim, glibc 계열 공식 이미지에서 런타임만 복사)
COPY --from=node:24.19.0-trixie-slim@sha256:ab3eebe934147fee049b5eb83c570f68c849a13c930bdfa482de99fcdfa3b3de \
    /usr/local/bin/node /usr/local/bin/node
COPY --from=node:24.19.0-trixie-slim@sha256:ab3eebe934147fee049b5eb83c570f68c849a13c930bdfa482de99fcdfa3b3de \
    /usr/local/lib/node_modules /usr/local/lib/node_modules
RUN ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
    ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx && \
    node --version | grep -qx 'v24.19.0' || (echo "ERROR: node version mismatch" && exit 1) && \
    npm --version | grep -qx '12.0.2' || (echo "ERROR: npm version mismatch, expected 12.0.2" && exit 1)

# Git + Playwright Chromium Headless Shell 구동에 필요한 최소 OS 패키지
RUN apt-get update && apt-get install -y --no-install-recommends \
        git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# @playwright/test 자체는 App Lock으로 CI 단계에서 설치한다(Lock 변경 금지 원칙).
# 여기서는 App package.json에 고정된 버전(1.63.0)과 동일한 CLI로 Chromium Headless Shell +
# 필요 시스템 라이브러리만 Workspace 바깥에 미리 굽는다.
RUN npx --yes playwright@1.63.0 install --with-deps chromium --only-shell \
    && mv /root/.cache/ms-playwright /opt/playwright-cache

RUN groupadd --gid 1000 ciuser && \
    useradd --uid 1000 --gid ciuser --shell /bin/bash --create-home ciuser && \
    chmod -R go-w /opt/playwright-cache   # root 소유 유지, 그룹/기타 쓰기 권한만 제거

USER ciuser
WORKDIR /home/workspace
