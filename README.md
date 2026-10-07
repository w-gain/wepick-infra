# wepick-infra

## 저장소 역할과 제품 설계

이 저장소는 **실행 환경·배포·운영**를 관리합니다. 제품·정책·ERD·화면 정의서·와이어프레임·공통 API 설계의 기준은 [wepick-product](https://github.com/W-Gain/wepick-product)입니다. [문서 관리 규칙](https://github.com/W-Gain/wepick-product/blob/main/docs/guides/repository-and-document-guide.md)을 따르며 설계 원본을 복사하지 않습니다.

## 현재 상태

- **운영 배포 환경은 없습니다.** 기존 AWS 리소스는 삭제했고, 2026-10-07에 AWS·self-hosted runner 배포 자산과 모든 배포 워크플로를 제거했습니다.
- 이 저장소는 **목표 런타임 구성**(Compose, Caddy, 환경 계약)만 관리합니다.
- 배포 방식(호스트, 이미지 전달, 외부 진입 경로)은 [Product 전환 계획](https://github.com/W-Gain/wepick-product/blob/main/docs/plans/2026-09-target-product-transition.md) 8단계에서 정합니다. 진행 상태는 [계획 현황](https://github.com/W-Gain/wepick-product/blob/main/docs/plans/README.md)을 봅니다.

## 목표 런타임

```text
Browser
  └─ Caddy :80/:443 (automatic TLS)
      ├─ /api/*     → backend:8080
      ├─ /uploads/* → uploads_data volume (read-only)
      └─ /*         → frontend

backend
  ├─ mysql:3306 (internal only)
  └─ uploads_data:/data/uploads
```

자세한 구성과 환경 계약은 [런타임 구조](docs/host-architecture.md)를 봅니다.

## 구성

| 경로 | 내용 |
|---|---|
| `compose/prod` | Caddy, frontend, backend, MySQL 런타임 구성 |
| `caddy` | 외부 라우팅과 TLS 설정 |
| `environments/prod/.env.example` | 런타임 환경변수 이름과 예시 값 |
| `scripts/deploy-host.sh` | 환경 파일로 Compose를 검사·기동하고 backend health를 확인 |
| `scripts/rollback-host.sh` | 이전 환경 파일로 `deploy-host.sh` 재실행 |
| `docs/host-architecture.md` | 런타임 구조, 환경 계약, 미결정 사항 |
| `docs/history` | 과거 운영 기록 (현재 검증 결과 아님) |

## 검증

```bash
docker compose --env-file environments/prod/.env.example -f compose/prod/docker-compose.yml config -q
docker run --rm -e DOMAIN_NAME=wepick.example.com -e ACME_EMAIL=admin@example.com \
  -v "$PWD/caddy/Caddyfile:/etc/caddy/Caddyfile:ro" \
  caddy:2.10-alpine caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

## 문서

- [런타임 구조와 환경 계약](docs/host-architecture.md)
- [HQ에서 이전한 2026-08-20 운영 기록](docs/history/hq-2026-08-20.md) — 현재 운영 검증 결과가 아님
- [제품·공통 시스템 설계](https://github.com/W-Gain/wepick-product)
- [작업 지침](AGENTS.md)
