# wepick-infra

## 저장소 역할과 제품 설계

이 저장소는 **실행 환경·배포·운영**를 관리합니다. 제품·정책·ERD·화면 정의서·와이어프레임·공통 API 설계의 기준은 [wepick-product](https://github.com/W-Gain/wepick-product)입니다. [문서 관리 규칙](https://github.com/W-Gain/wepick-product/blob/main/docs/guides/repository-and-document-guide.md)을 따르며 설계 원본을 복사하지 않습니다.

아래 구현 설명은 기존 구현에 관한 기록이며 최신 제품 요구사항을 대신하지 않습니다. 현재 동작은 코드·검증 결과로 확인하고, 목표와의 차이는 [Product 전환 작업](https://github.com/W-Gain/wepick-product/blob/main/docs/plans/documentation-backlog.md)에 연결합니다.

Wepick의 단일 운영 저장소입니다. 프론트엔드와 백엔드는 GitHub-hosted Actions CI로 테스트된 불변 이미지를 GHCR에 발행하고, 이 저장소의 self-hosted Actions runner가 production CD를 실행합니다.

## 현재 운영 목표

```text
GitHub-hosted Actions (FE/BE CI) → GHCR immutable images
                                      ↓
GitHub Actions self-hosted runner (infra CD) → Docker Compose host
                                              ├─ Caddy (80/443, automatic TLS)
                                              ├─ frontend
                                              ├─ backend
                                              └─ MySQL named volume
```

- 운영 런타임: 단일 Docker host
- 프록시: Caddy
- 이미지 registry: GHCR
- CD 실행자: self-hosted GitHub Actions runner
- 비밀값: 초기에는 GitHub Environment Secrets와 호스트의 미추적 runtime environment file, 추후 HashiCorp Vault 검토
- 데이터베이스: infra Compose가 독립 MySQL service와 named volume을 소유

## Layout

| Path | Purpose |
|---|---|
| `compose/prod` | Caddy, frontend, backend, MySQL production topology |
| `caddy` | Public routing and TLS configuration |
| `environments/prod` | Immutable image and runtime environment templates |
| `runner` | Self-hosted runner security and bootstrap guide |
| `scripts` | Host deployment and rollback commands |
| `docs/host-architecture.md` | Current host deployment decisions and runtime contract |
| `docs/first-host-deployment.md` | M0 deployment prerequisites and production smoke checklist |
| `terraform`, `docker`, `nginx`, legacy workflows | Historical AWS deployment assets; not the host deployment target |

## Validation

```bash
docker compose --env-file environments/prod/.env.example -f compose/prod/docker-compose.yml config -q
docker run --rm -e DOMAIN_NAME=wepick.example.com -e ACME_EMAIL=admin@example.com \
  -v "$PWD/caddy/Caddyfile:/etc/caddy/Caddyfile:ro" \
  caddy:2.10-alpine caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

## AWS legacy

AWS resources are not destroyed in this branch. Terraform, EC2/SSM/ECR delivery scripts, and existing AWS workflows remain preserved for historical reference. Legacy AWS workflows are manual-only and are not part of host CD.

## 문서

- [제품·공통 시스템 설계](https://github.com/W-Gain/wepick-product)
- [Host 구조와 환경 계약](docs/host-architecture.md)
- [최초 배포·검증 절차](docs/first-host-deployment.md)
- [Runner 안내](runner/README.md)
- [HQ에서 이전한 2026-08-20 운영 기록](docs/history/hq-2026-08-20.md) — 현재 운영 검증 결과가 아님
- [작업 지침](AGENTS.md)
