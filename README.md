# MCP Google Search Server — TECH GUIDE

| https://github.com/ssong-openmaru-io/noapi-google-search-mcp-proxy-server

## 1. Overview

본 문서는 Playwright 기반 Google 검색 MCP 서버를 컨테이너 환경에서 안정적으로 실행하기 위한 기술 가이드입니다.

해당 서버는 다음 구성요소를 기반으로 동작합니다.
- noapi-google-search-mcp : Google 검색 기능을 제공하는 MCP 서버
- mcp-proxy : STDIO 기반 MCP 서버를 HTTP/SSE 인터페이스로 변환
- Playwright Chromium : 브라우저 자동화를 통한 검색 수행

본 이미지는 다음 환경에서 사용하도록 설계되었습니다.
- Flowise Custom MCP
- Docker Standalone MCP
- Kubernetes MCP Gateway
- AI Agent Platform Integration

⸻

## 2. Architecture
```
Client / LLM
        │
        ▼
HTTP (Port 8000)
        │
        ▼
mcp-proxy
        │  (STDIO)
        ▼
noapi-google-search-mcp
        │
        ▼
Playwright Chromium
        │
        ▼
Google Search
```

핵심 특징:
- STDIO MCP → HTTP 변환 지원
- Headless Chromium 기반 검색
- Non-root 보안 실행
- 브라우저 재다운로드 방지
- 컨테이너 환경 최적화

## 3. Dockerfile

아래 Dockerfile은 Playwright 브라우저 경로 문제와 권한 문제를 해결하고,
MCP Proxy 환경에서도 안정적으로 동작하도록 설계된 운영용 구성입니다.
```
# ------------------------------------------------------------
# Base Image
# ------------------------------------------------------------
# Microsoft 공식 Playwright Python 이미지 사용
# - Chromium / Firefox / WebKit 브라우저가 /ms-playwright 경로에 사전 설치되어 있음
# - pwuser 비권한 사용자 포함
FROM mcr.microsoft.com/playwright/python:v1.58.0-jammy


# ------------------------------------------------------------
# Working Directory
# ------------------------------------------------------------
WORKDIR /app


# ------------------------------------------------------------
# Python Environment Setup
# ------------------------------------------------------------
# MCP 서버 및 proxy 설치
RUN pip install --upgrade pip && \
    pip install noapi-google-search-mcp mcp-proxy


# ------------------------------------------------------------
# Playwright Browser Cache Preparation
# ------------------------------------------------------------
# Playwright 기본 탐색 경로:
#   ~/.cache/ms-playwright
#
# 공식 이미지 브라우저 위치:
#   /ms-playwright
#
# pwuser 홈 캐시 위치로 복사하여:
# - 권한 문제 방지
# - 환경변수 의존성 제거
# - MCP Proxy 환경 안정성 확보
RUN mkdir -p /home/pwuser/.cache && \
    cp -r /ms-playwright /home/pwuser/.cache/ms-playwright && \
    chown -R pwuser:pwuser /home/pwuser/.cache


# ------------------------------------------------------------
# Environment Variables
# ------------------------------------------------------------
ENV PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
    PYTHONUNBUFFERED=1


# ------------------------------------------------------------
# Security: Non-root User
# ------------------------------------------------------------
USER pwuser


# ------------------------------------------------------------
# Network
# ------------------------------------------------------------
EXPOSE 8000


# ------------------------------------------------------------
# Entrypoint
# ------------------------------------------------------------
ENTRYPOINT ["mcp-proxy"]
CMD ["--port=8000", "--host=0.0.0.0", "--", "noapi-google-search-mcp"]
```

## 4. Docker Image Build

### 4.1 Build Command
```
docker build -t mcp-google-search -f Dockerfile.server .
```
### 4.2 Run Command
```
docker run -p 8000:8000 mcp-google-search
```

## 5. API Endpoint
서버 실행 후 MCP Endpoint:
```
http://localhost:8000/sse
```

## 6. Flowise Integration Example

Flowise Custom MCP 설정 예시:
```
{
  "name": "google-search",
  "url": "http://10.20.1.10:8000/sse"
}
```

## 7. Kubernetes Deployment Example
### Deployment:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-google-search
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-google-search
  template:
    metadata:
      labels:
        app: mcp-google-search
    spec:
      containers:
        - name: mcp
          image: mcp-google-search:latest
          ports:
            - containerPort: 8000
```

### Service:

```
apiVersion: v1
kind: Service
metadata:
  name: mcp-google-search
spec:
  selector:
    app: mcp-google-search
  ports:
    - port: 80
      targetPort: 8000
```

## 8. Kubernetes Service Endpoint

Kubernetes 환경에서 서비스가 배포되면,
클러스터 내부에서는 다음과 같은 Service DNS Endpoint로 접근할 수 있습니다.

```
http://mcp-google-search.default.svc.cluster.local/sse
```
| 항목|설명|
|---|---|
|Service Name|mcp-google-search|
|Namespace|default|
|Cluster Domain|svc.cluster.local|

네임스페이스가 변경된 경우:
```
http://mcp-google-search.<namespace>.svc.cluster.local/sse
```

## 9. Referer
- https://discuss.pytorch.kr/t/noapi-google-search-mcp-google-search-api-key-mcp/8968
- https://www.piwheels.org/project/noapi-google-search-mcp/
- https://github.com/sparfenyuk/mcp-proxy
- https://playwright.dev/python/docs/docker