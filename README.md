# BlogCabin 문서

블로그들이 모여 사는 오두막 마을, **BlogCabin(블로그캐빈)**의 문서 저장소다. 작성·관리: 정지원(jiwon934).
코드는 따로 [`cabinlog1070/blogcabin`](https://github.com/cabinlog1070/blogcabin) 저장소에 있다.

| 문서 | 내용 |
|---|---|
| [`docs/01-requirements.md`](docs/01-requirements.md) | 요구사항 명세 (기능 AUTH·BLOG·POST·SOC·GAME·SHOP·TOWN, 비기능 NF) |
| [`docs/02-erd.md`](docs/02-erd.md) | 지금 구현된 DB 설계 (코드의 마이그레이션 기준) |
| [`docs/erd/`](docs/erd/) | 처음 Crowfoot으로 설계한 ERD와 DDL (요구사항 v1.8 시점) |
| [`specs/`](specs/) | 영역별 기능 명세 7개 (Spec Kit) |
| [`.specify/memory/constitution.md`](.specify/memory/constitution.md) | 개발 원칙 |

실행·배포 방법은 코드 저장소의 `README.md`와 `docs/03-deploy.md`에 있다.

요구사항과 ERD는 **이 저장소에서만** 고친다(2026-10-14부터, 코드 저장소에는 사본을 두지 않는다). 기능을 만들며 요구사항이나 DB 설계가 바뀌면 코드 PR과 함께 이 저장소에도 PR을 올린다. 이 저장소는 `main` 브랜치 하나로 운영한다: 브랜치를 만들어 고치고 `main`으로 PR.
