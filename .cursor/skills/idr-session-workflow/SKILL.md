---
name: idr-session-workflow
description: >-
  Enforces the IDR lis_cursor session workflow (Gate A–E), mandatory document
  read order, and CURRENT/plan.md hygiene. Use at session start, before coding or
  running tests, when editing docs/CURRENT_WORK_SESSION.md, docs/plans/plan.md,
  or docs/rules in this repository.
---

# IDR 세션 워크플로 (에이전트용 요약)

**전체 규정(단일 원본)**: `docs/rules/workflow_gates.md` — 상충 시 그 문서가 우선한다.

## 세션 시작 시 읽기 순서

1. `docs/plans/plan.md` — Phase·WBS 위치
2. `docs/CURRENT_WORK_SESSION.md` — 현재 Gate, 구현/테스트 계획
3. `docs/rules/error_analysis.md` — 알려진 AI 오판
4. 구현·리뷰 시: `docs/rules/project_context.md`, `docs/rules/backend_architecture.md`, `ref_files/IDR_Data_Analysis_SDD.md` (필요 범위)

## Gate A ~ E (한 줄)

| Gate | 할 일 | 승인 전 금지 |
|------|--------|----------------|
| **A** | `CURRENT`에 구현 상세 계획 (**`docs/plans/` 신설로 대체 금지**) | 코드 변경 |
| **B** | 구현 후 구현 완료 요약·`[x]` | Gate C/D 테스트 착수 |
| **C** | `CURRENT`에 테스트 계획(상세) (**`docs/plans/` 신설 금지**) | 테스트 작성·실행 |
| **D** | 실행·검증 결과 기록 | (실패 시 재실행 후 기록) |
| **E** | `WORK_HISTORY.md`, `plan.md` 갱신, `CURRENT` 다음 세션용 교체 | E 생략 |

## Gate B 직후 필수 멈춤 (가장 자주 위반됨)

**Gate A만 승인된 상태**(예: 사용자가 계획에 「진행해라」)에서 구현을 끝낸 뒤:

1. `CURRENT`에 **구현 완료 요약(Gate B)** 만 작성하고 진행 상태를 **`구현 완료 — 사용자 확인 대기`** 로 둔다.
2. **여기서 에이전트 턴을 종료한다.** `make test`·pytest·Gate C/D/E를 **시작하지 않는다**.
3. 사용자가 구현을 검토하고 **별도로** 테스트 진행을 허용하면(또는 처음부터 「테스트까지」명시하면) 그때 Gate C → D → E를 진행한다.

이 단계를 건너뛰면 `docs/rules/error_analysis.md` 「Gate B 직후 멈춤 없이…」 항목에 해당하는 위반이다.

## ga-server MCP (`user-ga-server-ssh`) — **ga-nginx(Docker) 설정만**

**사용자가 문서로 고정한 뜻**: ga-server MCP는 **서버 위 Docker로 떠 있는 엣지 nginx(예: `ga-nginx`) 안의 설정**을 **확인**하고, 사용자가 **명시적으로** nginx 설정 수정을 요청한 경우에만 **그 범위**를 손본다.

| 하지 않음 (반복 오류) | 하기 |
|----------------------|------|
| `idr-fastapi`·Postgres·Redis 등 **다른 컨테이너** `docker exec` 로 404 «해결» | `docker exec ga-nginx …` 로 **conf 읽기**만 (기본) |
| 호스트 `lis_cursor` 경로에 파일 업로드·compose·빌드 | 앱·정적 문제는 **리포 + 로컬 운영형 / public-edge 절차** |
| nginx가 이미 맞는데 스니펫만 반복 수정 | `{"detail":"Not Found"}` → **§0** 기준 로컬 터널·로컬 **`demo/ide`** |
| 공인 404 를 **`idr-fastapi` 재빌드·compose** 가 기본 해결로 안내 | **금지** — 운영 표준은 **`lis_public_url_path_map.md` §0** (오직 로컬 우회) |

상세: `docs/rules/project_context.md` 「ga-server·공인 URL·MCP」. **§0·경로 표**: `docs/plans/lis_public_url_path_map.md`. 사례: `docs/rules/error_analysis.md` MCP·§0 항목.

## AI 금지 (요약)

- A 승인 없이 구현 시작.
- **A 승인만으로 구현 후 곧바로 테스트 실행(D) 또는 이력/세션 마감(E)까지 진행하기** — 반드시 B 확인 대기 후 C 승인.
- B 승인 없이 테스트 계획(C) 또는 실행(D).
- C 승인 없이 테스트 코드·실행.
- D 기록 없이 이력만 갱신하거나 `CURRENT`를 다음 세션으로 통째 교체.
- E에서 `plan.md` 체크·진행 표 누락.

## 상세 계획 문서 위치 (반복 위반 방지)

- **Gate A·C의 상세**(실행 순서, Phase 표, DoD, 체크리스트)는 **`docs/CURRENT_WORK_SESSION.md`에만** 기록한다.
- **`docs/plans/<새파일>.md`를 세션 상세 계획용으로 만들지 않는다.** `docs/plans/`는 `plan.md`, 경로 정본(`lis_public_url_path_map.md` 등), 강의·패키지 **참조**용이다.
- 사용자가 「상세 계획 작성」만 요청해도 → **`CURRENT` 해당 섹션을 채운다** (별도 plans 신설 아님). 위반 시 `docs/rules/error_analysis.md`에 기록.

## `CURRENT_WORK_SESSION.md` 권장 섹션 순서

메타·워크플로 표 → 구현 상세 계획(A) → 구현 완료 요약(B) → 테스트 계획(C) → 검증 결과(D). 상세 템플릿은 `workflow_gates.md` §표준 섹션.

## 품질·커밋 (pre-commit 충돌 방지)

- 커밋 전: `make format && make lint && make typecheck` 통과 후 변경분 **전부** `git add` (unstaged 남기지 않기 — stash 롤백으로 훅 수정이 폐기될 수 있음).
- 상세: `docs/rules/error_analysis.md` (pre-commit·E501·mypy 재발 항목).

## 코드·아키텍처 힌트 (자주 씀)

- Dify는 수치 계산 안 함 — FastAPI/Pandas에서 계산.
- `endpoints/*.py`에 `from __future__ import annotations` 금지 (422 이슈).
- 분석 API에 `compact=true` 지원.
- 엔드포인트·서비스 메서드는 async 일관성.

## 관련 경로

- 이력: `docs/history/WORK_HISTORY.md`
- Git 커밋 한국어: `.cursor/rules/git-commit-korean.mdc`, `.vscode/settings.json` (Copilot 보조)
