# BlogCabin 운영 배포 안내


## 0. 지금 쓰는 배포 방식: GitHub Actions + Docker + SSH (2026-10-08)

`main`에 올리면 `.github/workflows/deploy.yml`이 자동으로 배포한다 (Actions 화면의 **Run workflow**로 직접 실행해도 된다).

1. 서버에 SSH로 접속해 CPU 종류(amd64/arm64)와 Docker 사용 가능 여부를 확인한다
2. 그 CPU에 맞는 Docker 이미지 두 개를 빌드해 파일 하나로 묶는다: 화면 `blogcabin-web`(`frontend/Dockerfile`), 서버 `blogcabin-api`(`backend/Dockerfile`)
3. 이미지 파일·`deploy.env`(DB 주소 등)·`deploy/deploy.sh`를 서버의 `~/blogcabin/releases/`로 보낸다
4. 서버에서 `deploy/deploy.sh`가 이미지를 불러오고 → DB 마이그레이션(서버 이미지) → `db:seed` → (첫 배포만) `admin:create` (이 둘은 화면 이미지) → 새 이미지를 시험용 포트(8449)에서 먼저 켜 보고 → **롤링 배포**로 앱 두 개를 하나씩 새 이미지로 바꾼다 (아래). 실패하면 직전 이미지로 되돌린다

### 무중단 롤링 배포: 포트 두 개 (2026-10-15)

```text
https://blogcabin.java21.net ─▶ 서버 관리자 nginx(443) ─▶ upstream blogcabin_java21_app (~/nginx/blogcabin.java21.net.conf)
                                                           ├─▶ 앱 blogcabin-1 (:8440)
                                                           └─▶ 앱 blogcabin-2 (:8441)   ← 번갈아(라운드로빈)
```

- 앱 하나가 화면과 서버를 같이 받는다. 화면(Next.js)이 `/api/v1/…` 요청을 서버 앱(`backend/`의 Hono)에 그대로 넘겨 안에서 돌린다(`frontend/src/app/api/v1/[...path]/route.ts`). 로그인 `/api/auth/…`는 화면의 같은 로그인 설정이 받는다. 서버 코드는 `backend/`에 그대로 있고, 서버 이미지(`blogcabin-api`)는 배포 때 DB 마이그레이션에만 쓴다
- 앞의 https nginx는 여러 학생이 같이 쓰는 서버 관리자 것이다. 그 nginx가 각 계정의 `~/nginx/*.conf`를 읽어서, 우리 파일 `~/nginx/blogcabin.java21.net.conf`에 upstream(8440·8441, `max_fails=1 fail_timeout=5s`)과 `proxy_next_upstream error timeout http_502 http_503`을 넣었다. 한쪽이 멈추면 nginx가 다른 쪽으로 다시 보낸다(이중화)
- 이 파일은 `deploy.sh`가 upstream이 없을 때 **한 번만** 고친다: Certbot 줄은 그대로 두고, nginx 문법 검사를 통과할 때만 바꾸고, 원래 파일은 `~/blogcabin/nginx-backup/`에 남긴다. 첨부(30MB)가 들어오게 `client_max_body_size`를 40M로 올린다. 틀린 줄이 들어가면 모든 학생의 nginx가 다시 읽기를 못 하니 손으로 고칠 때도 조심한다. upstream 이름은 다른 학생과 겹치지 않게 길게 지었다
- 고친 파일은 **서버 nginx가 다시 읽어야(reload)** 적용된다. 다시 읽기는 관리자 권한이라 우리 계정으로는 못 하지만, 서버 nginx가 **10분마다(매시 00·10·20…분) 알아서 다시 읽는다**(2026-10-15 09:20:00·09:30:00에 다시 읽은 것을 확인). 그래서 파일을 고치면 늦어도 10분 안에 적용되고, 선생님께 따로 부탁하지 않아도 된다. 적용 전까지 서버 nginx는 옛 설정만 알아서, 그 사이 배포하면 바꾸는 몇 초 동안 끊길 수 있다
- 배포 순서: ① 새 이미지를 시험용 포트 8449에서 켜서 상태 확인(화면 `/api/health`와 서버 `/api/v1/health` 둘 다)을 통과하는지 본다. 서버 nginx에는 우리가 `down`을 붙일 수 없어서 새 앱이 상태 확인 전에도 손님을 받기 때문에, 망가진 이미지는 여기서 거른다(통과 못 하면 사이트는 하나도 안 바뀐다) → ② 2번(8441)을 멈추고 새 이미지로 바꿔 상태 확인 → 8초 기다린다 → ③ 1번(8440)을 멈추고 바꾼다
- 8초를 기다리는 까닭: 서버 nginx는 멈췄던 2번을 5초(`fail_timeout`) 동안 "죽음"으로 기억한다. 그 사이 1번까지 멈추면 갈 곳이 없어 502가 난다(시험으로 확인). 이 몇 초 동안은 옛 버전과 새 버전이 같이 손님을 받는다
- ②에서 새 버전이 안 뜨면 2번을 직전 이미지로 되돌린다. ③에서 안 뜨면 두 앱을 모두 직전 이미지로 되돌린다(두 버전이 섞인 채 남지 않게)
- 앱을 멈출 때는 처리 중인 요청을 마칠 시간을 준다(`docker stop -t 30`). 서버 nginx는 접속이 안 되는 쪽을 실패로 보고 다른 쪽으로 다시 보낸다
- **API·DB 바꾸기 주의**: 배포하는 동안 옛 버전과 새 버전이 잠깐 같은 DB를 쓴다. 칼럼이나 API 주소를 지우거나 이름을 바꿀 때는 "새 것 추가 → 다음 배포에서 옛 것 정리"처럼 두 번에 나눈다
- 두 앱은 같은 DB, 같은 `~/blogcabin/uploads`를 쓴다. 로그인은 DB에 저장돼서 어느 앱으로 가도 그대로다
- 예전 구성(우리 nginx 컨테이너 8440 → 화면 8441·8442, 서버 8443·8444, 2026-10-14)에서 처음 바꿀 때는 예전 nginx가 8441로 보내지 않게 한 뒤 새 앱을 8441에 켜고, 8440을 예전 nginx에서 새 앱으로 넘긴다. 그 순간만 1초 안팎 끊긴다. 8440 새 앱이 안 뜨면 예전 nginx를 다시 켠다. 중간에 끊긴 배포를 다시 돌려도 이어서 한다

| 저장소 시크릿 | 쓰임 |
|---|---|
| `SSH_ADDRESS`, `SSH_PORT`, `SSH_ID`, `SSH_PASSWORD` | 배포 서버 SSH 접속 (비밀번호 방식) |
| `DB_ADDRESS`, `DB_PORT`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD` | `DATABASE_URL`로 합쳐서 컨테이너에 넘김 (비밀번호의 특수문자는 자동 인코딩) |
| `APP_URL` (선택) | 사용자가 여는 주소. 없으면 `http://SSH_ADDRESS:8440`. 도메인·https를 붙이면 꼭 넣는다 (로그인 쿠키·로그아웃이 이 주소 기준) |
| `TRUSTED_ORIGINS`, `DB_SSLMODE`, `GOOGLE_*`, `KAKAO_*`, `NAVER_*` (선택) | 추가 허용 주소, DB SSL, 소셜 로그인 |

서버 쪽 준비와 폴더:
- Docker가 설치돼 있고, SSH 계정이 `docker`를 쓸 수 있어야 한다 (docker 그룹 또는 비밀번호 없는 sudo)
- `DB_ADDRESS`가 `localhost`이거나 기본 네트워크로 DB에 닿지 않으면 앱이 서버 네트워크를 그대로 쓴다(`--network host`). 앱은 늘 0.0.0.0에서 듣는다: 127.0.0.1로 듣게 하면 블로그 주소(`/@아이디`)가 500이 된다(2026-10-14 핫픽스). 아니면 `-p 127.0.0.1:8440:8440`처럼 서버 안에서만 열린다
- `~/blogcabin/secrets.env`: 첫 배포 때 서버가 만든 로그인 비밀키(`BETTER_AUTH_SECRET`)와 관리자 비밀번호(`ADMIN_PASSWORD`). 지우면 모든 회원이 로그아웃되니 지우지 않는다. 관리자 비밀번호 보기: `grep ADMIN_PASSWORD ~/blogcabin/secrets.env`
- `~/blogcabin/uploads/`: 글 첨부 사진·파일 (컨테이너의 `/data/uploads`). 백업 대상
- `~/blogcabin/nginx-backup/`: `deploy.sh`가 서버 nginx 설정을 고치기 전의 원래 파일
- 로그 보기: `docker logs --tail 100 blogcabin-1` (`blogcabin-2`). 서버 nginx 로그는 관리자 것이라 볼 수 없다

## 1. 구조: 화면(frontend)과 서버(backend)

저장소 하나에 폴더 두 개다. 원래 **Next.js 16 풀스택 앱 하나**(지금의 `frontend/`)가 화면도 그리고 DB 일도 했고, 2026-10-14부터 DB 일을 영역별로 `backend/`(Node + Hono, 주소 `/api/v1/…`)로 옮기는 중이다. 아래 표의 코드 위치는 `frontend/` 기준이다.

| 폴더 | 하는 일 | 상태 확인 |
|---|---|---|
| `frontend/` | 화면. 아직 옮기지 않은 영역은 Server Action으로 DB를 직접 쓴다. 아이템 시드·관리자 계정도 여기. 운영에서는 `/api/v1/…`도 서버 앱을 안에서 돌려 받는다 | `/api/health` |
| `backend/` | 서버 API `/api/v1/…`, 로그인 `/api/auth/…`. DB 코드·마이그레이션도 여기. 개발 때는 따로 띄우고(4000), 운영에서는 화면 안에서 돈다 | `/api/v1/health` |

| 역할 | 어디서 도나 | 코드 |
|---|---|---|
| 화면(프론트) | 서버 컴포넌트는 Node 서버에서 HTML로 그리고, 클라이언트 컴포넌트(에디터, 꾸미기 끌어다 놓기, 농장 등)는 브라우저에서 돈다 | `src/app/**/page.tsx`, `src/components/` |
| 광장 게임 | 브라우저에서만 도는 Phaser 4 (서버 일 없음) | `src/components/town/` |
| 서버 로직 | 같은 Node 서버의 **Server Action** (글 저장, 구매, 출석 등) | `src/app/**/actions.ts`, `src/server/` |
| API | 같은 Node 서버의 Route Handler: 로그인 `/api/auth/*`, 첨부 올리기 `/api/uploads`, 첨부 보기 `/files/[키]` | `src/app/api/`, `src/app/files/` |
| 요청 앞단 | `src/proxy.ts` (예전 middleware): 초대 링크 쿠키 등 | `src/proxy.ts` |
| DB | PostgreSQL (Drizzle ORM) | `backend/src/db/schema.ts`, `backend/drizzle/` |
| 첨부 파일 | **서버 디스크** 폴더 (`UPLOAD_DIR`, 기본 `storage/uploads`) | `src/server/storage.ts` |

```text
브라우저 ──HTTPS──▶ 서버 nginx ─┬─▶ 앱 1 (:8440) ─┐  앱 = 화면(frontend, next start)
                                └─▶ 앱 2 (:8441) ─┤        + 그 안의 서버(backend) /api/v1/*
                                                  ├─▶ PostgreSQL
                                                  └─▶ 디스크 폴더 UPLOAD_DIR (사진·파일)
```

## 2. 운영에 필요한 것

| 항목 | 내용 |
|---|---|
| Node.js | 24 (개발은 v24.21.0) |
| PostgreSQL | 16 이상 권장 (개발은 17). 운영 DB는 매일 백업 (NF-24) |
| 디스크 | 첨부 파일용 **지워지지 않는 디스크(볼륨)**. 서버를 다시 배포해도 남아 있어야 한다 |
| HTTPS | 필수. 로그인 쿠키가 운영에서는 `Secure`로만 전송된다 (NF-11) |
| 도메인 | 예: `https://blogcabin.example.com`. 소셜 로그인 콜백 주소에 쓰인다 |

### 호스팅 고르기

- **추천: 디스크를 붙일 수 있는 컨테이너·VM 호스팅** (Render, Railway, Fly.io, 또는 클라우드 VM + Docker). 지금 코드 그대로 올라간다.
- **Vercel 같은 서버리스**: 화면과 서버 로직은 돌아가지만 **서버 디스크가 남지 않아서 첨부 사진·파일이 사라진다.** 쓰려면 먼저 `src/server/storage.ts`를 S3·Cloudflare R2 같은 저장소로 바꾸는 작업이 필요하다.
- DB는 같은 곳의 관리형 PostgreSQL, Neon, Supabase, 또는 지금 쓰는 Crowfoot 관리형 DB 중 하나.

## 3. 환경 변수 (운영 서버에 설정)

| 이름 | 값 | 비고 |
|---|---|---|
| `DATABASE_URL` | `postgres://사용자:비밀번호@호스트:5432/DB이름` | 관리형 DB면 `?sslmode=require`가 필요할 수 있다 |
| `BETTER_AUTH_SECRET` | `openssl rand -base64 32` 로 새로 만든 값 | 개발 값을 재사용하지 않는다 |
| `BETTER_AUTH_URL` | 운영 주소, 예: `https://blogcabin.example.com` | 끝에 `/` 없이. 쿠키·콜백·CSRF 확인에 쓰인다 |
| `ADMIN_USERNAME` | `admin` 등 | |
| `ADMIN_PASSWORD` | 12자 이상 추측하기 어려운 값 | NF-14 |
| `UPLOAD_DIR` | 볼륨 경로, 예: `/data/uploads` | 비우면 `storage/uploads` (재배포 때 사라질 수 있음) |
| `GOOGLE_CLIENT_ID` / `_SECRET` | 구글 클라우드 콘솔 | 비우면 그 버튼이 비활성 |
| `KAKAO_CLIENT_ID` / `_SECRET` | 카카오 디벨로퍼스 | 〃 |
| `NAVER_CLIENT_ID` / `_SECRET` | 네이버 개발자센터 | 〃 |

비밀 값은 호스팅 서비스의 환경 변수 화면에만 넣는다. `.env.local`은 Git에 올리지 않는다 (NF-01).

### 소셜 로그인 콜백 주소 (각 개발자 센터에 등록)

- 구글: `https://도메인/api/auth/callback/google`
- 카카오: `https://도메인/api/auth/callback/kakao`
- 네이버: `https://도메인/api/auth/callback/naver`

## 4. 배포 순서

Docker 없이 직접 올릴 때 (지금은 위 0번의 Actions 방식을 쓴다). 화면은 `frontend/`, 서버는 `backend/`에서 실행한다.

처음 한 번:

```bash
npm ci                  # 저장소 맨 위에서 (화면·서버 둘 다)
cd frontend
npm run build
npm run db:migrate      # backend/drizzle/ 마이그레이션을 운영 DB에 적용
npm run db:seed         # 아이템·동물 카탈로그 (여러 번 해도 안전)
npm run admin:create    # 관리자 계정·공지 블로그
npm run start -- -p 3000
cd ../backend && npm run build && PORT=4000 npm start   # 서버 (/api/v1/…)
```

그다음 배포마다: 맨 위에서 `npm ci`, `frontend/`에서 `npm run build && npm run db:migrate && npm run start`, `backend/`에서 `npm run build && npm start`. 앞에 nginx를 두고 `/api/v1/`·`/api/auth/`를 서버(4000)로 보낸다
(아이템을 바꿨으면 `npm run db:seed`도)

- `npm run db:reset`은 **개발 DB를 비우는 명령이라 운영에서 절대 쓰지 않는다.**
- DB 구조는 마이그레이션 파일로만 바꾼다 (NF-22).
- 서버는 하나만 띄운다. 첨부가 디스크에 있어서 서버를 여러 대로 늘리면 파일이 나뉜다 (늘리려면 S3·R2로 바꾼 뒤).

## 5. 배포 뒤 확인

- [ ] `https://도메인` 첫 화면이 열리고, 아이디 회원가입 → 온보딩 → 광장까지 간다
- [ ] 로그인 응답 쿠키에 `HttpOnly; Secure; SameSite=Lax`가 있다 (NF-11)
- [ ] 글에 사진을 올리고, 서버를 다시 시작해도 사진이 보인다 (볼륨 확인)
- [ ] 소셜 로그인 3가지가 각각 된다
- [ ] 휴대폰에서 광장 메뉴, 글쓰기, 꾸미기 끌어다 놓기, 농장이 된다
- [ ] `/feed`가 1초 안에 보인다 (NF-07, `e2e/nonfunctional.mjs`)
- [ ] DB 자동 백업이 켜져 있다 (NF-24)

## 6. 아직 정할 것

- 호스팅 서비스와 DB 서비스
- 도메인
- 첨부를 디스크에 둘지, S3·R2로 바꿀지 (서버리스나 서버 여러 대를 쓰려면 바꿔야 함)
- `main`에 올리면 자동 배포할지 (NF-08), PR마다 자동 검사(CI)를 돌릴지 (NF-23)
