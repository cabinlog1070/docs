# Specification Quality Checklist: 회원 / 인증

**Purpose**: Validate specification completeness and quality before proceeding to planning
**Created**: 2026-10-07
**Feature**: [spec.md](../spec.md)

## Content Quality

- [x] No implementation details (languages, frameworks, APIs)
- [x] Focused on user value and business needs
- [x] Written for non-technical stakeholders
- [x] All mandatory sections completed

## Requirement Completeness

- [ ] No [NEEDS CLARIFICATION] markers remain
- [x] Requirements are testable and unambiguous
- [x] Success criteria are measurable
- [x] Success criteria are technology-agnostic (no implementation details)
- [x] All acceptance scenarios are defined
- [x] Edge cases are identified
- [x] Scope is clearly bounded
- [x] Dependencies and assumptions identified

## Feature Readiness

- [x] All functional requirements have clear acceptance criteria
- [x] User scenarios cover primary flows
- [x] Feature meets measurable outcomes defined in Success Criteria
- [x] No implementation details leak into specification

## Notes

- Items marked incomplete require spec updates before `/speckit-clarify` or `/speckit-plan`
- 남은 `[NEEDS CLARIFICATION]` 3개 (`/speckit-clarify`에서 결정):
  1. FR-008 (AUTH-07): 비밀번호 복구 방법 — 이메일 선택 입력 + 재설정 / 관리자 초기화 / 이번 범위에서 제공 안 함
  2. FR-030 (AUTH-06): 탈퇴한 회원이 남의 글에 단 댓글(과 거기 달린 답글)을 함께 삭제할지, "탈퇴한 회원"/`삭제된 댓글이에요`로 남길지
  3. FR-036 (AUTH-05): 아이디와 소셜로 따로 가입해 생긴 두 계정을 허용할지, 하나로 합치는 방법을 제공할지
- FR-030·FR-036의 해당 부분, 그리고 User Story 6 시나리오 3은 위 결정 전까지 확정된 수용 기준이 없다. 이 항목들은 "All functional requirements have clear acceptance criteria"에서 결정 대기로 간주했다.
- 알려진 문제로 표시한 FR: FR-014(로그인 시도 제한이 실제 로그인 경로에 적용되지 않을 수 있음, NF-10), FR-035(소셜 연결 키 미발급).
- 사용자 문구(오류·안내)와 `blogville/@` 같은 화면 표시값은 제품 요구사항이라 원문 그대로 두었다. 구현 경로·함수·테이블 이름은 본문에 넣지 않았다.
- 원천 문서 AUTH 영역에 `❌ 제외` 항목은 없다.
