<!--
Sync Impact Report
- Version change: (template) → 1.0.0
- Modified principles: 템플릿 자리표시자 5개 → 아래 7개 원칙으로 처음 정의
  I. 글쓰기가 먼저, 게임은 돕는 역할
  II. 권한과 검증은 서버에서
  III. 돈처럼 다루는 데이터의 무결성
  IV. 저장할 때 정화하고, 비밀은 환경 변수에
  V. 확인할 수 있는 수용 기준
  VI. 요구사항 ID는 바꾸지 않는다
  VII. 모바일까지 쓸 수 있는 한국어 화면
- Added sections: 기술 제약, 개발 흐름과 품질 관문, Governance
- Removed sections: 없음
- Templates: plan/spec/tasks 템플릿은 실행 시점에 이 문서를 읽으므로 수정하지 않음
- Deferred TODOs: 없음. 비준일은 요구사항 문서 v0.1 작성일(2026-10-02)로 둠
- Source: docs/01-requirements.md v1.8 (1장 개요, 4.0 작성 양식, 5장 비기능 요구사항)
-->

# BlogCabin Constitution

## Core Principles

### I. 글쓰기가 먼저, 게임은 돕는 역할

BlogCabin은 블로그 활동을 게임 보상과 연결해 꾸준히 쓰게 만드는 서비스다.

- 글쓰기와 꾸미기가 제품의 중심이다. 레벨·코인·광장·동물 농장 같은 게임 요소는 글쓰기를
  돕는 역할에 머물러야 하며(MUST), 글을 쓰고 읽는 흐름을 막아서는 안 된다(MUST NOT).
- 코인은 활동으로만 얻는다. 유료 결제는 만들지 않는다(MUST NOT).
- 이번 범위 밖: 실시간 멀티플레이, 유료 결제, 모바일 앱(반응형 웹으로 대응).

근거: 요구사항 1.2 목표 1번, 1.3 범위 밖.

### II. 권한과 검증은 서버에서

- 글 수정·삭제, 장착, 구매, 출석, 관리자 기능 등 상태를 바꾸는 모든 처리는 서버에서
  권한을 검사한다(MUST). 모든 Server Action은 첫 줄에서 `requireMember()` 또는
  `requireAdmin()`을 호출한다.
- 입력 검증은 서버(zod)가 기준이다. 브라우저 검증은 편의일 뿐 보안 경계가 아니다.
- 다른 사이트에서 온 요청(CSRF)은 거부한다(MUST). Server Action이 아닌 Route Handler는
  `Origin`과 `Host`를 직접 비교해 다르거나 없으면 403을 돌려준다.
- 로그인 쿠키는 HttpOnly, SameSite=Lax, 배포 환경에서 Secure이어야 한다(MUST).

근거: NF-02, NF-11, NF-12.

### III. 돈처럼 다루는 데이터의 무결성

- 코인·경험치·레벨은 별도 칸에 저장하지 않고 원장(ledger) 합계로 계산한다(MUST).
- 코인 지급·차감, 아이템 구매처럼 원장을 바꾸는 처리는 하나의 트랜잭션으로 묶는다(MUST).
  중간에 실패하면 전부 취소된다.
- "한 글에 공감 한 번", "출석은 하루 한 번", "보유한 아이템만 장착" 같은 규칙은 앱 코드와
  함께 DB 제약조건(UNIQUE, CHECK, FK)으로도 막는다(MUST). 동시에 두 번 요청해도 한 번만
  처리되어야 한다.
- DB 구조 변경은 마이그레이션 파일로만 한다(MUST). DB를 직접 고치지 않는다.

근거: NF-04, NF-05, NF-15, NF-16, NF-22.

### IV. 저장할 때 정화하고, 비밀은 환경 변수에

- 에디터로 쓴 HTML은 저장할 때 허용 태그만 남기고 정화한다(MUST). `<script>`, `on*` 속성,
  `javascript:` 링크, `<iframe>`, `<style>`은 남지 않으며, 사진·파일은 우리 저장소 주소만
  허용한다.
- DB 쿼리는 ORM의 값 바인딩으로만 만든다(MUST). 입력값을 SQL 문자열에 이어 붙이지 않는다.
- 비밀번호는 해시로만 저장하고 원문을 DB·로그·화면에 남기지 않는다(MUST).
- DB 접속 정보, 로그인 비밀키, 소셜 Client Secret, 관리자 비밀번호는 서버 환경
  변수(`.env.local`)에만 둔다(MUST). 저장소에 커밋하지 않는다.
- 개인정보는 서비스에 필요한 최소한(아이디, 소셜 닉네임·프로필 사진)만 저장한다(SHOULD).

근거: NF-01, NF-03, NF-09, NF-13, NF-26.

### V. 확인할 수 있는 수용 기준

- 요구사항 하나에는 기능 하나만 담는다(MUST).
- 수용 기준은 측정하거나 눈으로 확인할 수 있게 쓴다(MUST). "빠르게" 대신 "1초 안에",
  "잘 보인다" 대신 "375px에서 가로 스크롤이 없다".
- 수용 기준은 그대로 테스트 시나리오가 된다. 각 기준에는 확인 방법(E2E 스크립트, 단위 테스트,
  코드 리뷰, 수동 확인)을 붙인다(SHOULD).
- 모르는 것은 지어내지 않고 열린 질문(spec에서는 `[NEEDS CLARIFICATION]`)으로 남긴다(MUST).
  팀이 정한 뒤에 본문으로 옮긴다.
- 상태는 실제로 확인한 결과만 완료(✅)로 표시한다(MUST).

근거: 요구사항 4.0 작성 양식, 5장 머리말.

### VI. 요구사항 ID는 바꾸지 않는다

- `AUTH-01`, `POST-07`, `NF-12` 같은 요구사항 ID는 한 번 정하면 바꾸지 않는다(MUST).
  spec, plan, tasks, 코드, 테스트가 이 ID로 서로를 가리킨다.
- 없앤 요구사항은 지우지 않고 상태를 `❌ 제외`로 바꾼다(MUST).
- Spec Kit 산출물(spec.md, plan.md, tasks.md)은 근거가 된 요구사항 ID를 함께 적는다(MUST).

근거: 요구사항 문서 "이 문서를 함께 고치는 방법" 4번.

### VII. 모바일까지 쓸 수 있는 한국어 화면

- 모든 회원 화면은 375px 너비에서 가로 스크롤 없이 쓸 수 있어야 한다(MUST).
- 버튼·링크의 누르는 영역은 최소 44×44px이고(SHOULD), 버튼 글자는 두 줄로 쪼개지지 않는다.
- 사용자에게 보여 주는 문구는 한국어로 쓰고, 오류 문구는 무엇을 고치면 되는지 알려 준다(SHOULD).
  보여 줄 문구는 spec에 그대로 적어 화면과 서버가 같은 말을 쓰게 한다.
- 키보드(Tab, Enter)만으로 로그인·글쓰기·댓글·구매를 할 수 있어야 한다(SHOULD).

근거: NF-06, NF-18, NF-19.

## 기술 제약

현재 구현을 기준으로 한 기술 스택이며, 바꾸려면 plan에서 근거를 밝히고 이 문서를 개정한다.

- 웹: Next.js App Router, 서버 컴포넌트 우선, 상태 변경은 Server Action
  (파일 업로드처럼 필요한 경우만 Route Handler)
- 데이터: 관계형 DB + Drizzle ORM, 마이그레이션은 `drizzle/` 폴더
- 인증: Better Auth (사이트 아이디 + 네이버·카카오·구글 소셜 로그인)
- 검증: zod / 에디터: Tiptap / HTML 정화: sanitize-html / 스타일: Tailwind CSS
- 광장(게임 화면): Phaser 4, 브라우저에서만 동적으로 불러온다
- 그림 에셋: 코드로 그린 창작 SVG(`src/lib/art/`). 외부 에셋은 상업적 이용이 허용된 것만 쓰고
  출처를 남긴다(NF-25)
- 성능 목표: 글 목록 화면 1초 안에 표시(NF-07), 광장 약 60fps(NF-20)
- 호환성 목표: 최신 Chrome, Edge, Safari, 모바일 Chrome·Safari(NF-21)

## 개발 흐름과 품질 관문

- 기능은 Spec Kit 흐름을 따른다: `/speckit-specify` → (`/speckit-clarify`) →
  `/speckit-plan` → `/speckit-tasks` → (`/speckit-analyze`) → `/speckit-implement`.
- 원천 요구사항은 `docs/01-requirements.md`다. spec과 원천 문서가 다르면 원천 문서를 먼저
  고치고 변경 이력에 한 줄 남긴다.
- `main`에서 브랜치를 만들고, PR은 다른 팀원 1명 이상이 승인한 뒤 merge한다.
- merge 전에 타입 검사·린트·관련 E2E를 통과해야 한다(NF-23).
- 배포 전 관리자 비밀번호를 12자 이상, 추측하기 어려운 값으로 바꾼다(NF-14).

## Governance

- 이 constitution은 다른 개발 관행보다 우선한다. plan의 Constitution Check는 위 원칙을
  기준으로 통과 여부를 적고, 어긴다면 Complexity Tracking에 이유를 남긴다.
- 개정은 PR로 하며, 팀원 1명 이상의 승인이 필요하다. 개정할 때는 Sync Impact Report를 갱신한다.
- 버전 규칙: 원칙을 없애거나 뜻을 바꾸면 MAJOR, 원칙·절을 더하거나 크게 넓히면 MINOR,
  문구만 다듬으면 PATCH.
- 코드 리뷰는 원칙 II~IV(서버 권한 검사, 트랜잭션·DB 제약, 정화·비밀 관리)를 반드시 확인한다.

**Version**: 1.0.0 | **Ratified**: 2026-10-02 | **Last Amended**: 2026-10-07
