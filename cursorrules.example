# Cursor AI 지시문 — IDR 시스템 데이터 분석 AI 백엔드

**[Git / SCM — 최우선]** 소스 제어에서 AI가 **생성하는 커밋 메시지는 반드시 한국어**로만 작성한다 (영어 제목·본문 금지). 형식은 Conventional Commits: `feat(범위): 한글 제목`. (세부 타입·예시는 `.cursor/rules/git-commit-korean.mdc` 및 `.vscode/settings.json`의 Copilot 커밋 지시와 동일 계열로 맞출 것.)

## 필수 지시사항

Cursor AI야, 이 프로젝트에서 코드를 작성하거나 아키텍처를 결정하기 전에
반드시 아래 규칙 문서들을 먼저 읽고 모든 내용을 반영해라.
문서를 읽지 않고 코드를 작성하는 것은 금지한다.

---

## 작업 문서 체계 (세션 시작 시 반드시 확인)

### 전체 계획서
`docs/plans/plan.md`

- 프로젝트 전체 Phase(0~7) WBS. 마일스톤 단위 체크리스트.
- 현재 어느 Phase까지 완료되었는지 파악하기 위해 세션 시작 시 읽어라.
- **코드 작업 전 반드시 이 문서로 현재 위치를 확인할 것.**

### 현재 작업 세션
`docs/CURRENT_WORK_SESSION.md`

- **단위 작업(태스크) 시작 전 반드시 이 문서를 먼저 읽어라.**
- 현재 진행 중인 태스크의 **상세 계획**(구현 순서, Phase·DoD, 체크리스트, 주의사항)은 **`docs/CURRENT_WORK_SESSION.md`에만** 기록한다. **`docs/plans/`에 세션 전용 상세 계획 마크다운을 신설하지 않는다** — `docs/plans/`는 `plan.md`·경로 정본·강의 참조 등 WBS·배경용이다 (`workflow_gates.md`·`SKILL.md` 「상세 계획 문서 위치」).
- 태스크가 완료되면 해당 항목을 `- [x]`로 체크하고, 상세 완료 내역은 `docs/history/WORK_HISTORY.md`로 이전한다.
- **구현·테스트 게이트 A~E (필수)**: 순서·금지 사항·`CURRENT` 섹션 규정은 **`docs/rules/workflow_gates.md`** 가 단일 원본이다. 에이전트용 압축 요약은 **`.cursor/skills/idr-session-workflow/SKILL.md`** 를 병행한다. `plan`/참조 확인 후 코딩 전 CURRENT에 구현 상세 계획·사용자 승인; 단계 생략·승인 생략 금지. **Gate A 승인(예: 「진행해라」)은 구현(Gate B)까지만** — 구현 직후 **`make test`/Gate C·D·E는 별도 승인** 없이 진행하지 말 것(SKILL.md 「Gate B 직후 필수 멈춤」).
- 진행 중에는 표준 섹션(워크플로 상태, 구현 요약, 테스트 계획, 검증 결과)을 유지한다. **사용자 승인을 받은 뒤에만** "다음 세션 전용 내용만 남은 상태"로 정리한다.

### 작업 이력
`docs/history/WORK_HISTORY.md`

- 완료된 태스크의 상세 내역(구현 결정 사항, 변경 파일 목록, 특이사항)을 누적 기록한다.
- `CURRENT_WORK_SESSION.md`에서 완료 처리된 태스크를 이 문서로 옮겨 기록한다.
- 형식: `## [날짜] <태스크명>` → 완료 내용 요약.

### 오류 분석 기록
`docs/rules/error_analysis.md`

- **AI가 잘못 판단하여 오류를 발생시킨 사례를 즉시 이 문서에 기록한다.**
- 새 세션 시작 시 이 문서를 읽고 동일한 실수를 반복하지 않는다.
- 형식: 오류 유형 / 발생 상황 / 근본 원인 / 재발 방지 규칙.
- `backend_architecture.md`에 이미 있는 패턴과 중복되더라도 이 문서에도 기록한다 (맥락 보존).

---

## 규칙 문서 목록

### 1. 프로젝트 컨텍스트
`docs/rules/project_context.md`

- 프로젝트 정체성, 비즈니스 목표(SCM/CRM/BI), 기술 스택 전체
- 디렉토리 구조, 실행 환경(RHEL 8 + podman), 작업 프로토콜(Gate A~E)
- **ga-server·MCP(`user-ga-server-ssh`)**: **Docker `ga-nginx`(엣지 nginx) 컨테이너 안 설정** 확인·(사용자가 명시 시) 그 conf 수정만. **`idr-fastapi`·호스트 파일 쓰기·compose·배포는 MCP로 하지 않는다.** 공인 `lis.*` **FastAPI 업스트림 운영 표준은 오직 로컬 우회** — **`docs/plans/lis_public_url_path_map.md` §0** 필독. **`/`·`/apps`·`/ide/`** 경로 역할은 동 파일 정본. `/ide` JSON 404 시 **먼저** nginx·로컬 터널·로컬 `demo/ide`; **`idr-fastapi` 이미지 재빌드를 운영 표준 처방으로 제시하지 않는다.** `project_context.md`·`error_analysis.md` MCP·§0 관련 항목.
- **Git · 추적 제외 및 비밀 관리** — **`.env*` 전부 ignore**(예외 없음). 템플릿은 `env.example`·`env.prod.example`·`infra/dify/env.vendor.example` 등 **점 없는 파일명**. `git check-ignore` 확인.
- **코드를 작성하기 전 반드시 이 문서를 먼저 읽어라**

### 2. 백엔드 아키텍처 규칙
`docs/rules/backend_architecture.md`

- async/await 기반 API 작성 규칙 및 주의사항
- 2-Tier 하이브리드 라우팅 패턴 (Pandas Tier 1 vs Dify Tier 2)
- FastAPI Depends 기반 의존성 주입(DI) 패턴
- Pandas 서비스 계층 분리 원칙
- pre-commit / ruff / mypy 품질 게이트 규칙
- **이전 프로젝트에서 실제로 발생한 오류 패턴과 재발 방지 규칙 포함**

### 3. Dify 연동 규칙
`docs/rules/dify_integration.md`

- FastAPI ↔ Local Self-hosted Dify 역할 분리 원칙
- `compact=true` 응답 포맷 표준 (LLM 컨텍스트 절약)
- Dify HTTP Request Node에서 파싱하기 쉬운 JSON 구조 설계 가이드
- Docker 네트워크 공유 및 포트 구성

### 4. 오류 분석 기록
`docs/rules/error_analysis.md`

- AI 오판 사례 및 재발 방지 규칙 누적 기록
- **세션 시작 시 반드시 읽어라. 문서가 없으면 아직 기록된 오류가 없는 것이다.**

### 5. 구현·테스트 게이트 (워크플로)
`docs/rules/workflow_gates.md` (전문) · `.cursor/skills/idr-session-workflow/SKILL.md` (요약)

- **세션 시작 시** `CURRENT`의 워크플로 상태(어느 Gate인지)를 확인한다.

---

## 핵심 원칙 요약 (빠른 참조용)

1. **워크플로 게이트 A~E 준수**: 단계·금지 사항은 `docs/rules/workflow_gates.md` 및 `.cursor/skills/idr-session-workflow/SKILL.md` 따름. 단계 생략 금지. **구현 완료 후 Gate B에서 멈춤** — 테스트는 C 승인 후에만.
2. **Dify는 데이터를 직접 처리하지 않는다**: 모든 수치 계산은 FastAPI에서.
3. **endpoints/*.py에서 `from __future__ import annotations` 사용 금지**: 422 에러 원인.
4. **`compact=true` 파라미터**: 모든 분석 엔드포인트에 필수 구현.
5. **async/await 일관성**: 모든 엔드포인트와 서비스 메서드는 async.
6. **Spring Boot 비유로 설명**: 개발자 백그라운드가 Spring Boot / Kotlin / JPA / Flyway.
7. **오류 발생 시 즉시 기록**: AI 오판이 확인되면 `docs/rules/error_analysis.md`에 즉시 추가.

---

## 참조 SDD

전체 아키텍처 결정 근거와 API 명세는 아래 문서에 있다:
`ref_files/IDR_Data_Analysis_SDD.md` (v1.1.0)
