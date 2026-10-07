# 런타임 구조

- 마지막 수정: 2026-10-07
- 상태: 목표 구성. 현재 운영 배포 환경은 없음

## 구성

단일 Docker host에서 Caddy, frontend, backend, MySQL을 Docker Compose로 실행합니다. Caddy만 호스트 포트(80/443)를 열고 TLS를 자동 발급합니다. backend와 MySQL은 호스트 포트를 열지 않습니다.

- `/api/*`: Caddy가 경로의 `/api`를 떼고 `backend:8080`으로 프록시
- `/uploads/*`: backend가 쓰는 `uploads_data` 볼륨을 Caddy가 `/srv/uploads`에 읽기 전용으로 마운트해 제공
- `/*`: frontend

backend는 이미지를 `ImageStorage` 인터페이스 뒤의 로컬 볼륨(`/data/uploads`)에 저장합니다. S3·MinIO 전환은 이 인터페이스 안에서 결정합니다.

## 환경 계약

Compose는 `--env-file`로 받은 환경 파일을 사용합니다. Git에는 이름과 예시 값만 두며 기준은 [`environments/prod/.env.example`](../environments/prod/.env.example)입니다.

| 변수 | 용도 |
|---|---|
| `DOMAIN_NAME`, `ACME_EMAIL` | Caddy 사이트 주소와 TLS 발급 |
| `FE_IMAGE`, `FE_IMAGE_TAG`, `BE_IMAGE`, `BE_IMAGE_TAG` | 실행할 애플리케이션 이미지 |
| `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` | MySQL 초기화와 backend DB 접속 |

backend에 필요한 나머지 값은 Compose가 위 값으로 만들어 넣거나(`SPRING_DATASOURCE_*`, `CORS_ALLOWED_ORIGINS`), 고정값(`SESSION_COOKIE_SECURE=true`)으로 넣습니다. 그 밖의 backend 변수는 기본값을 사용합니다. 목록은 [BE README](https://github.com/W-Gain/wepick-be#runtime-environment-variables)를 봅니다.

## 실행

```bash
./scripts/deploy-host.sh <환경 파일 경로>
```

Compose 구성을 검사하고, 이미지를 받고, 서비스를 기동한 뒤 backend `/actuator/health`를 확인합니다. 되돌릴 때는 이전 이미지 태그가 담긴 환경 파일로 `scripts/rollback-host.sh`를 실행합니다.

## 데이터 보호

- `docker compose down -v`를 운영 데이터에 실행하지 않습니다.
- `wepick_mysql_data`와 `wepick_uploads_data`는 유지해야 하는 named volume입니다.
- 자동 백업은 아직 없습니다.

## 미결정 사항 (8단계)

- **frontend 제공 방식:** Caddyfile은 `frontend:3000`으로 프록시하지만, 현재 FE 운영 이미지는 자체 Caddy로 Vite `dist/`를 80번에서 제공합니다. 이대로는 연결되지 않습니다. 목표는 단일 Caddy가 `dist/`를 직접 제공하는 구조입니다([ADR-0002](https://github.com/W-Gain/wepick-product/blob/main/docs/decisions/0002-frontend-technology-stack.md)).
- **배포 경로:** 운영 호스트, 이미지 전달 방식(로컬 빌드·레지스트리), 배포 실행 방식
- **외부 진입:** 80/443 직접 포트 포워딩 또는 Cloudflare Tunnel
- **백업:** MySQL·업로드 볼륨의 외부 백업과 복구 연습
