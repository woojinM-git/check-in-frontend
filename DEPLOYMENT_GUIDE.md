# Frontend CI/CD 가이드

프런트엔드는 Next.js 기반 정적/SSR 혼합 애플리케이션으로, nginx 리버스 프록시와 함께 별도 EC2에서 구동합니다. 백엔드와 마찬가지로 GitHub Actions가 품질 검증과 배포까지 전담하며, 두 파이프라인은 독립적으로 운영됩니다.

---

## 아키텍처 개요
- 프런트엔드와 백엔드를 분리 배포한다. 프런트는 전용 EC2 인스턴스에서 Docker Compose로 `frontend`(Next.js)와 `nginx` 컨테이너를 함께 띄운다.
- GitHub Actions가 `main` 브랜치 push 시 빌드/테스트를 수행하고, 성공하면 동일 워크플로에서 SSH로 EC2에 접속해 최신 이미지를 재배포한다.
- Cloudflare → Nginx → Next.js 경로로 요청이 흐르며, nginx가 백엔드 API(`/api/**`, `/oauth2/**`)를 프록시해 쿠키를 공유한다.

---

## 사전 준비
- **EC2 인스턴스**
  - OS: Ubuntu 22.04 LTS 기준
  - 패키지: `git`, `docker`, `docker compose plugin`
  - 배포 디렉터리: `/home/ubuntu/check-in-frontend` (GitHub Actions에서 `FRONTEND_PROJECT_PATH`로 사용)
    - 최초 1회만 아래 명령으로 생성  
      ```
      sudo mkdir -p /home/ubuntu/check-in-frontend
      sudo chown ubuntu:ubuntu /home/ubuntu/check-in-frontend
      git clone https://github.com/<ORG>/check-in-frontend.git /home/ubuntu/check-in-frontend
      ```
  - SSL 인증서: Cloudflare Origin 인증서를 `./ssl/`에 저장해 nginx 컨테이너에 마운트
- **Cloudflare**
  - DNS, WAF, SSL Full 모드 설정
  - `checkinn.store`, `www.checkinn.store` A 레코드를 프런트 EC2로 지정
- **GitHub Secrets**
  - `FRONTEND_SSH_HOST`, `FRONTEND_SSH_PORT`(선택), `FRONTEND_SSH_USER`, `FRONTEND_SSH_KEY`
  - `FRONTEND_PROJECT_PATH` (예: `/home/ubuntu/check-in-frontend`)
  - `FRONTEND_ENV_FILE` : `.env.production` 전체 내용
- **환경 변수 (.env.production)**
  - `NEXT_PUBLIC_API_URL`, `BACKEND_HOST`, `NEXT_PUBLIC_TOSS_CLIENT_KEY`, `TOSS_SECRET_KEY`
  - `NEXT_PUBLIC_KAKAO_MAP_API_KEY`, `NEXT_TOUR_KEY`, `NEXT_PUBLIC_ROOM_IMAGE_BASE_URL`
  - `ENABLE_HTTPS_REDIRECT=true` (프로덕션)

---

## GitHub Actions 워크플로
- 파일 경로: `.github/workflows/frontend-cicd.yml`

### 빌드 Job (`build`)
1. `actions/checkout@v4`
2. `actions/setup-node@v4` / Node 18
3. `npm ci`
4. `npm run lint`
5. `npm run build`
6. `actions/upload-artifact@v4`로 `.next`, `package.json`, `package-lock.json` 저장(3일)

### 배포 Job (`deploy`)
1. 빌드 결과 아티팩트를 내려받아 기록
2. `appleboy/ssh-action@v1.0.0`으로 프런트 EC2 접속
3. 원격 스크립트
   - `git fetch --all && git reset --hard origin/main`
   - `.env.production`에 `FRONTEND_ENV_FILE`을 그대로 쓰기
   - `docker compose pull --ignore-pull-failures`
   - `docker compose up -d --build frontend nginx`
   - `docker image prune -f`

---

## Docker 구성
- `Dockerfile`
  - Node 18 Alpine 베이스
  - `npm ci`, `npm run build` 후 `npm start`
  - `NODE_ENV=production`, `NEXT_TELEMETRY_DISABLED=1`
- `docker-compose.yml`
  - `frontend` 서비스: 위 이미지 빌드, `NODE_ENV`/`NEXT_PUBLIC_API_URL` 전달
  - `nginx` 서비스: `default.conf.template`와 `ssl` 디렉터리 마운트, 80/443 노출
  - 공통 네트워크 `checkin_net`
- **초기 세팅 체크리스트**
  - `/home/ubuntu/check-in-frontend/.env.production` 작성(Secrets와 동일 내용)
  - `/home/ubuntu/check-in-frontend/docker-compose.yml`이 최신인지 `git pull`로 동기화
  - `./ssl/certificate.crt`, `./ssl/private.key` 파일 존재 여부 확인

---

## 수동 배포 절차
1. `ssh -i <pem> ubuntu@<host>`
2. `cd /home/ubuntu/check-in-frontend`
3. `git fetch --all && git reset --hard origin/main`
4. 필요 시 `nano .env.production`
5. `docker compose pull --ignore-pull-failures`
6. `docker compose up -d --build frontend nginx`
7. `docker compose logs -f frontend` 혹은 `nginx`로 상태 점검

---

## 롤백 전략
- 릴리스마다 태그(`vYYYYMMDD`)를 남기고, 문제가 발생하면 해당 태그로 체크아웃 후 동일한 Compose 명령을 수행한다.
- 또는 `docker image ls`로 이전 이미지 ID를 확인해 `docker compose up -d --build` 대신 `docker compose up -d`로 구 버전을 재기동한다.
- 필요 시 `/home/ubuntu/check-in-frontend/releases` 디렉터리를 두고 `docker save` 혹은 `git archive`로 빌드 산출물을 보관한다.

---

## 모니터링 및 점검
- 헬스체크 엔드포인트(예: `/api/tour/health`)를 Cloudflare 혹은 외부 모니터에 등록
- `docker compose logs frontend|nginx -f`, `docker stats`, `htop`으로 자원 확인
- GitHub Actions 실패 시 Slack/Teams 웹훅 알림 연동 권장

---

## 배포 체크리스트
- [ ] Secrets(`FRONTEND_*`) 최신 상태 확인
- [ ] `.env.production` 값 검증 (민감 정보 포함)
- [ ] `npm run lint`, `npm run build` 로컬 통과
- [ ] Cloudflare SSL 모드 및 DNS 확인
- [ ] 주요 페이지/결제/관리자 화면 Smoke Test

---

## 문제 해결 가이드
| 증상 | 확인 포인트 |
| --- | --- |
| `npm run build` 실패 | 환경 변수, API 호출 실패 로그 |
| 502 Bad Gateway | `frontend` 컨테이너 상태, `docker compose logs nginx` |
| OAuth Redirect 오류 | `BACKEND_HOST`, `X-Forwarded-Proto`, Cloudflare SSL 모드 |
| 정적 리소스 404 | `_next/static` 프록시 블록, 빌드 산출물 존재 여부 |

---

백엔드 CI/CD 가이드는 별도 문서를 참고하고, 프런트엔드는 본 문서 흐름을 따른다. 두 파이프라인 모두 GitHub Actions를 중심으로 운영해 동일한 개발/배포 경험을 유지한다.

