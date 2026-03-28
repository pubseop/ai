# AI 오류 분석 기록 — IDR 시스템 데이터 분석 AI 백엔드

> **목적**: AI가 잘못 판단하여 오류를 유발한 사례를 기록하고, 동일한 실수가 반복되지 않도록 재발 방지 규칙을 명시한다.
> **사용법**: 세션 시작 시 이 문서를 읽어라. AI 오판이 확인되면 즉시 아래 형식으로 추가한다.

---

## 기록 형식

```
## [YYYY-MM-DD] <오류 제목>

**오류 유형**: (타입 에러 / 로직 오류 / 설계 오판 / 환경 설정 오류 / 기타)
**발생 상황**: 어떤 작업을 하다가 발생했는가
**근본 원인**: 왜 AI가 잘못 판단했는가
**재발 방지 규칙**: 다음 번에 이 상황에서 반드시 해야 할 것 / 하지 말아야 할 것
**관련 파일**: 해당 오류와 연관된 파일 경로
```

---

## 기록된 오류

### [2026-03-26] 구현 직후 세션 문서 전환 및 테스트 게이트 생략

**오류 유형**: 프로세스 위반 / 워크플로 누락 (기타)

**발생 상황**: Session 01(Phase 0~1 스캐폴딩) 코드 구현이 끝난 뒤, `docs/CURRENT_WORK_SESSION.md`를 사용자 검토·테스트 계획·테스트 검증 없이 곧바로 Session 02(Phase 2) 내용으로 교체하고 `WORK_HISTORY.md`만 갱신함.

**근본 원인**: `.cursorrules`와 `project_context.md`에 "계획 → 승인 → 구현 → 승인 → 테스트" 원칙은 있었으나, **구현 완료를 `CURRENT_WORK_SESSION.md`에 남기는 단계**와 **사용자 확인 후 테스트 계획·실행**을 문서·게이트로 강제하는 규정이 없어 AI가 한 턴에 마감 처리함.

**재발 방지 규칙**:
1. 구현 종료 직후 **`CURRENT_WORK_SESSION.md`에 「구현 완료 요약」**을 쓰고 상태를 **사용자 확인 대기**로 둔다. 사용자 명시 승인 전에는 테스트 계획으로 넘어가지 않는다.
2. 테스트는 **계획을 문서에 기록**한 뒤 승인받고, 실행 결과를 **「테스트 검증 결과」**에 남긴다.
3. 위 순서는 **`docs/rules/workflow_gates.md`** 및 갱신된 `.cursorrules` / `project_context.md` 작업 프로토콜(Gate A~E)을 따른다.

**관련 파일**: `docs/rules/workflow_gates.md`, `.cursorrules`, `docs/rules/project_context.md`, `docs/CURRENT_WORK_SESSION.md`

---

### [2026-03-26] "작업 계획 작성" 요청 시 Cursor CreatePlan 도구 오용

**오류 유형**: 프로세스 위반 / 도구 오용 (기타)

**발생 상황**: 사용자가 플랜 모드에서 "다음 작업 계획을 작성해라"고 했을 때, `docs/CURRENT_WORK_SESSION.md`에 상세 구현 계획을 기록하지 않고 Cursor의 CreatePlan 도구를 사용해 별도 `.plan.md` 파일을 생성함.

**근본 원인**: "작업 계획"이라는 말을 Cursor 플랜 모드의 내장 CreatePlan 기능으로 해석함. 이 프로젝트에서는 `CURRENT_WORK_SESSION.md`가 유일한 작업 계획 기록 장소임을 인식하지 못하고 범용 도구를 사용함.

**재발 방지 규칙**:
1. 플랜 모드에서 "작업 계획 작성" 또는 유사한 요청이 오면 → **`docs/CURRENT_WORK_SESSION.md`** 의 해당 섹션(구현 체크리스트, 완료 기준, 상세 스펙)을 채운다.
2. Cursor CreatePlan 도구(`CreatePlan`)는 이 프로젝트에서 **사용하지 않는다**.
3. 작업 계획의 기록 위치: `CURRENT_WORK_SESSION.md` (상세 스펙) → `WORK_HISTORY.md` (완료 이력).

**관련 파일**: `docs/CURRENT_WORK_SESSION.md`, `docs/rules/workflow_gates.md`, `.cursorrules`

---

### [2026-03-26] pre-commit: 테스트 한 줄 초과(E501) 및 BIService `prev` 삼항 연산 mypy 오류

**오류 유형**: 정적 분석 / 품질 게이트 (ruff E501, mypy `operator`)

**발생 상황**: Git 커밋 시 `pre-commit`이 `ruff`, `ruff-format`, `mypy`에서 실패. `test_crud_base.py`·`test_scm_service.py`에서 한 줄이 120자 초과(E501). `bi_service.py`의 `regional_trend`에서 `growth = ... if prev is None ... else (cur - prev) / prev` 형태로 인해 mypy가 `else` 가지에서도 `prev`에 `None` 가능성을 남겨 `float`와 `None` 간 `-`, `/` 연산 오류를 냄.

**근본 원인**: (1) 테스트에서 `create(..., {"key": ...})` 호출을 한 줄에 나열. (2) 삼항 연산자만으로는 일부 mypy 설정에서 `prev`의 좁히기(narrowing)가 불충분함. (3) unstaged 파일이 있을 때 pre-commit이 stash 후 훅이 파일을 고치면 롤백되어 사용자가 “고쳤는데도 실패”처럼 보일 수 있음.

**재발 방지 규칙**:
1. **ruff line-length 120**: 긴 함수 호출·dict 리터럴은 여러 줄로 나누거나 `poetry run ruff format`으로 맞춘 뒤 커밋한다. 커밋 전 `poetry run ruff check idr_analytics`로 확인.
2. **mypy + Optional 누적 변수**: `prev`처럼 `None`으로 시작해 루프에서 갱신하는 값은 `prev: float | None = None`으로 표기하고, `yoy_comparison`과 같이 **`if prev is None or prev == 0: ... else: (cur - prev) / prev`** 형태의 명시 분기를 쓴다. 성장률 계산은 삼항 한 줄에 `None` 가능 변수와 산술을 섞지 않는다.
3. **pre-commit stash 충돌**: 커밋 전에 스테이징과 맞추려면 `git add -A` 후 커밋하거나, unstaged 변경을 정리해 훅 자동 수정과 stash 복원이 충돌하지 않게 한다.

**관련 파일**: `idr_analytics/tests/unit/test_crud_base.py`, `idr_analytics/tests/unit/test_scm_service.py`, `idr_analytics/app/services/analytics/bi_service.py`, `.pre-commit-config.yaml`

---

### [2026-03-26] (재발) 동일 Git 로그(E501·bi_service 33행 mypy) — stash 롤백·미스테이징

**오류 유형**: 정적 분석 / pre-commit 동작 (환경·워크플로)

**발생 상황**: 이전에 기록한 것과 **동일한** `vscode.git.Git` 로그가 다시 나옴: `test_crud_base.py:78`·`test_scm_service.py:78`이 **한 줄짜리** 긴 코드로 보이고, `bi_service.py:33`에서 삼항 연산 기반 `(cur - prev) / prev` mypy 오류. 그러나 저장소의 올바른 수정본은 이미 **여러 줄 dict / DataFrame**, `bi_service`는 **`prev: float | None` + if/else** 형태이다.

**근본 원인**: (1) 커밋 시 **Unstaged files detected** → pre-commit이 unstaged를 stash한 뒤 훅이 스테이징된 스냅샷만 고침. (2) **Stashed changes conflicted with hook auto-fixes... Rolling back fixes** → 훅이 적용한 ruff-format·ruff --fix 결과가 **폐기**되고, 워킹 트리는 긴 줄·구버전 코드로 남음. (3) 수정이 **커밋/스테이징에 포함되지 않은** 채 IDE만 갱신된 것으로 착각하기 쉬움.

**재발 방지 규칙** (커밋 전 **순서 고정**):
1. 터미널에서 `make format && make lint && make typecheck` (또는 동일하게 `poetry run ruff format idr_analytics/`, `ruff check`, `mypy idr_analytics/app`)를 실행해 **로컬에서 먼저** 통과시킨다.
2. 그다음 **`git add -u`** 또는 **`git add -A`** 로 이번 변경 파일을 **전부 스테이징**한다. 일부만 stage하고 나머지를 unstaged로 둔 채 커밋하지 않는다.
3. 로그에 **한 줄짜리** `create(db, {"name": ...})`가 보이면 워킹 트리가 구버전이거나 롤백된 것이므로, 1~2를 반드시 다시 수행한다.
4. `bi_service.regional_trend`에는 **삼항 연산으로 `(cur - prev) / prev`를 쓰지 않는다** (mypy `operator` 재발 방지).

**관련 파일**: `Makefile` (`format` 타깃), `idr_analytics/tests/unit/test_crud_base.py`, `idr_analytics/tests/unit/test_scm_service.py`, `idr_analytics/app/services/analytics/bi_service.py`, `.pre-commit-config.yaml`

### [2026-03-26] passlib + bcrypt 4.x 로그인 시 `ValueError` / `bcrypt.__about__`

**오류 유형**: 라이브러리 호환 (런타임)

**발생 상황**: 통합 테스트·로그인에서 `verify_password` 경로로 passlib bcrypt 백엔드 초기화 시 `ValueError: password cannot be longer than 72 bytes` 또는 `AttributeError: module 'bcrypt' has no attribute '__about__'`.

**근본 원인**: passlib 1.7.4와 bcrypt 4.x/5.x API가 완전히 맞지 않음.

**재발 방지 규칙**:
1. 비밀번호 검증·해시는 **`bcrypt.checkpw` / `bcrypt.hashpw`** (`app/core/security.py`의 `verify_password`, `hash_password`)를 사용한다.
2. 통합 테스트 시드도 `hash_password()`를 사용한다.
3. `pyproject.toml`에 `bcrypt` 직접 의존을 명시하고 메이저 업 시 로그인 스모크를 수행한다.

**관련 파일**: `idr_analytics/app/core/security.py`, `idr_analytics/tests/integration/conftest.py`, `pyproject.toml`

---

### [2026-03-26] ga-nginx 바인드 마운트가 갱신되지 않아 설정 일부만 적용됨

**오류 유형**: 환경 설정 / 컨테이너 (운영)

**발생 상황**: 호스트의 `go-almond.swagger.conf`를 늘렸는데 `docker exec ga-nginx` 안의 `default.conf` 줄 수가 더 짧고, `nginx -T`에 `lis`용 `listen 443` 블록이 없음. `lis` 요청이 예전처럼 apex `qk54r71z` 블록으로 매칭되어 `/android/`로 301.

**근본 원인**: 바인드 마운트된 파일을 갱신한 뒤에도 컨테이너가 이전 inode/캐시를 보는 경우가 있음(환경에 따라 상이).

**재발 방지 규칙**:
1. 배포 후 **`docker exec ga-nginx wc -l /etc/nginx/conf.d/default.conf`** 로 호스트 `wc -l`과 일치하는지 확인한다.
2. 불일치 시 **`docker restart ga-nginx`** 후 `nginx -t`·`nginx -s reload`로 재적용한다.

**관련 파일**: ga-server `docs/nginx/go-almond.swagger.conf`, 컨테이너 `ga-nginx`

---

### [2026-03-26] pre-commit: unstaged stash 롤백 + mypy `security.py` / `agent.py` (jose·bcrypt·pandas)

**오류 유형**: 정적 분석 / pre-commit·mypy strict (`no-any-return`, `redundant-cast`)

**발생 상황**: VSCode Git 로그(`vscode.git.Git`)에서 커밋 시 **`[WARNING] Unstaged files detected`** → pre-commit이 unstaged 변경을 stash한 뒤 ruff·ruff-format이 스테이징된 파일만 수정하고, **`Stashed changes conflicted with hook auto-fixes... Rolling back fixes`** 로 자동 수정이 되돌아감. 이후 ruff는 통과했으나 **mypy**가 `idr_analytics/app/core/security.py`의 `hash_password` / `verify_password` / `create_access_token`, `idr_analytics/app/api/v1/endpoints/agent.py`의 `_pandas_answer`(DataFrame·`to_json` 분기)에서 **`Returning Any from function declared to return "str"`** 또는 **`Redundant cast to "str"`** 등을 보고해 커밋이 막힘.

**근본 원인**: (1) **일부만 스테이징**된 채 커밋하여 pre-commit stash와 훅 자동 수정이 충돌. (2) **python-jose** `jwt.encode`가 타입 스텁에서 **`Any`** 로 남는 경우가 많아 `str` 반환 함수와 맞지 않음. (3) **bcrypt** `hashpw` / `checkpw`는 스텁·버전에 따라 이미 `bytes` / `bool` 로 잡히므로, 여기에 `cast`를 얹으면 오히려 **`redundant-cast`** 가 난다. (4) **pandas** `DataFrame.to_json()` 반환이 스텁에서 `Any` 로 잡히면 `no-any-return` 이 난다(환경에 따라 `str` 로 잡혀 직접 반환 가능).

**재발 방지 규칙**:
1. 커밋 전 **`git add -A`**(또는 이번에 올릴 변경 **전부** 스테이징). unstaged를 남긴 채 IDE에서만 커밋하지 않는다. (`docs/rules/project_context.md` 「Git · 추적 제외」·README Git 체크리스트와 동일.)
2. 커밋 전 터미널에서 **`make format && make lint && make typecheck`** 로 ruff·mypy를 **로컬에서 먼저** 통과시킨다.
3. **`create_access_token`**: `jwt.encode(...)` 결과만 **`cast(str, ...)`** 으로 `str` 에 맞춘다. `bcrypt` 쪽은 스텁이 구체 타입을 주면 **cast 없이** 반환한다(redundant-cast 방지).
4. **`_pandas_answer`**: mypy가 `to_json` 을 `Any` 로만 보면 **`cast(str, raw)[:50000]`** 등으로 좁힌다. 수정 후 반드시 **`PYTHONPATH=idr_analytics poetry run mypy idr_analytics/app --strict`** 로 전체 검증.
5. 로그에 **stash / rollback** 문구가 보이면, 원인이 미스테이징인지 확인하고 1→2 순서를 다시 수행한다.

**추가(재발, 동일 증상)**: **`mirrors-mypy` 격리 환경**에는 `pyproject.toml` 의 Poetry 의존성이 자동으로 들어가지 않는다. `bcrypt`·`pandas` 가 없으면 해당 API가 **Any** → `hash_password` / `verify_password` / `DataFrame.to_json` 에 **`no-any-return`**. 반면 **`types-python-jose`** 를 `additional_dependencies` 에 넣으면 `jwt.encode` 가 **str** 로 잡혀 **`cast(str, jwt.encode(...))`** 가 **`redundant-cast`** 가 된다. **조치**: `.pre-commit-config.yaml` 의 mypy 훅에 `bcrypt`, `pandas`, `pandas-stubs`, `numpy` 등 앱이 쓰는 패키지를 Poetry 와 맞춰 넣고, `args` 에 **`--config-file=pyproject.toml`** 를 써서 프로젝트 mypy 설정과 통일한다. JWT 반환은 **`return str(jwt.encode(...))`** 로 로컬(jose Any)·pre-commit(str) 모두에서 `str` 로 맞춘다(불필요한 `str(str)` 는 런타임 무해).

**관련 파일**: `idr_analytics/app/core/security.py`, `idr_analytics/app/api/v1/endpoints/agent.py`, `.pre-commit-config.yaml`, `Makefile`, `pyproject.toml` (mypy·types)

---

### [2026-03-26] GitHub Push Protection — Dify 예시 `SECRET_KEY=sk-…` 오탐 → `.env*` 전면 ignore 정책

**오류 유형**: 원격 저장소 규칙 / 시크릿 스캔 (GH013, OpenAI API Key 패턴)

**발생 상황**: `git push` 시 **Push cannot contain secrets**. Dify 업스트림 `vendor/.env.example` 의 `SECRET_KEY` 예시가 **`sk-` 로 시작**해 GitHub가 OpenAI 키로 오탐.

**근본 원인**: 플레이스홀더라도 **`sk-` 접두**만으로 차단. **`git push --force` 로는 해결되지 않음**(같은 blob 이 남으면 계속 거절).

**재발 방지 규칙**:
1. **이름이 `.env`로 시작하는 파일은 Git 에 올리지 않는다** (`.gitignore` 의 `.env*`, 예외 없음). FastAPI 템플릿은 **`env.example`**, Dify 는 **`infra/dify/env.vendor.example`** 등 점 없는 파일명으로 관리한다.
2. `vendor` 동기화 시 새 변수는 **`env.vendor.example` 에 반영**하고, `SECRET_KEY` 등 **`sk-` 로 시작하는 예시 금지**.
3. 차단된 커밋은 파일 수정 후 **`git commit --amend`** 또는 히스토리 수정 후 **일반 `git push`**.

**관련 파일**: `.gitignore`, `env.example`, `infra/dify/env.vendor.example`, `infra/dify/README.md`

---

> 이후 AI 오판이 발생하면 위 형식으로 이어서 추가하세요.

---

### [2026-03-27] VSCode Git 커밋 시 pre-commit auto-fix 롤백(unstaged 충돌) 재발

**오류 유형**: 개발 워크플로 / pre-commit 충돌 (기타)

**발생 상황**: VSCode Git 커밋 로그에서 `Unstaged files detected` 후 `ruff`·`ruff-format`이 파일을 수정했지만, 곧바로 `Stashed changes conflicted with hook auto-fixes... Rolling back fixes...`가 발생해 자동 수정이 되돌아감.

**근본 원인**: 커밋 시점에 staged/unstaged 변경이 섞여 있었고, pre-commit이 unstaged를 stash한 상태에서 훅 자동 수정 결과와 복원 패치가 충돌함. 결과적으로 훅이 고친 내용이 워킹트리에 남지 않아 같은 오류가 반복될 수 있음.

**재발 방지 규칙**:
1. 커밋 전에 반드시 `make format && make lint && make typecheck`를 먼저 실행해 로컬 상태를 정리한다.
2. 커밋 직전에는 `git add -A`로 이번 변경을 전부 스테이징하고, staged/unstaged 혼재 상태에서 IDE 즉시 커밋을 피한다.
3. 동일 로그 문구(특히 `Rolling back fixes`)가 보이면 커밋을 중단하고, 포맷/린트 재실행 후 재스테이징한다.

**관련 파일**: `.pre-commit-config.yaml`, `Makefile`, `docs/rules/error_analysis.md`

---

### [2026-03-27] Gate B 직후 멈춤 없이 Gate C/D(테스트)까지 일괄 수행

**오류 유형**: 프로세스 위반 / 워크플로 누락 (Gate B·C 승인 생략)

**발생 상황**: Session 11에서 사용자가 Gate A(구현 상세 계획)에 대해 「진행해라」로 승인한 뒤, 에이전트가 코드 구현에 이어 **곧바로** `make format`·`make test` 등 테스트 검증(Gate D)까지 실행하고 `WORK_HISTORY`·`CURRENT` 교체(Gate E)까지 수행함.

**근본 원인**:
1. **승인 범위 과대해석**: 「진행해라」를 *해당 Gate에서 허용된 다음 한 단계(구현)*만이 아니라 *세션 전체 완료*로 해석함.
2. **일반적인 코딩 보조 관성**: “구현 후 바로 테스트로 검증”이 일반적이라, 프로젝트의 **B→C 사이 필수 대기** 규정을 `workflow_gates.md`에 있음에도 우선순위에서 밀어냄.
3. **명시적 멈춤 규칙 부재**: 규정 문서에는 금지 사항이 있으나, 에이전트 스킬·요약에 **“구현 끝나면 반드시 사용자에게 멈추고 C 승인 요청”**이 한 블록으로 강조되지 않아 실행 단계에서 누락됨.

**재발 방지 규칙**:
1. 사용자가 **Gate A만** 승인한 경우(예: 「진행해라」「구현해」·계획에 대한 동의): **구현(Gate B)까지만** 수행하고, `CURRENT`에 구현 완료 요약을 쓴 뒤 **반드시 멈춘다**. 테스트 계획(C)·실행(D)은 **사용자가 별도로 승인**한 뒤에만 진행한다.
2. 테스트까지 한 턴에 허용되려면 사용자가 **명시**해야 한다(예: 「구현 후 `make test`까지」「C/D 포함해서 진행」).
3. 상세 금지·승인 범위 해석은 **`docs/rules/workflow_gates.md`** §승인 범위 해석·§AI 동작 금지를 따른다. 세션 작업 시 **`.cursor/skills/idr-session-workflow/SKILL.md`** 의 「Gate B 직후 필수 멈춤」을 먼저 적용한다.

**관련 파일**: `docs/rules/workflow_gates.md`, `.cursor/skills/idr-session-workflow/SKILL.md`, `.cursorrules`, `docs/rules/project_context.md`

---

### [2026-03-27] VSCode Git 로그(첨부 1-24행) 기준 pre-commit `ruff-format` 롤백 재확인

**오류 유형**: 개발 워크플로 / staged-unstaged 혼재 (pre-commit stash 충돌)

**발생 상황**: 사용자 첨부 로그(`@vscode.git.Git` 1~24행)에서 커밋 시 `Unstaged files detected` → `ruff-format`이 1개 파일을 수정했으나 `Stashed changes conflicted with hook auto-fixes... Rolling back fixes...`로 되돌림. 현재 워킹트리에서도 대상 파일이 `AM` 상태(스테이징됨 + 미스테이징 변경 존재)로 확인됨.

**근본 원인**: 커밋 시점에 동일 파일 또는 연관 파일의 staged/unstaged 변경이 섞여 pre-commit의 임시 stash 복원과 auto-fix 결과가 충돌. 훅이 고친 내용이 유지되지 않아 같은 실패 로그가 반복됨.

**재발 방지 규칙**:
1. 커밋 직전 **한 번에 정리**: `make format && make lint && make typecheck`.
2. 그 다음 `git add -A`로 변경 전체를 재스테이징해 `AM`/혼재 상태를 제거한 뒤 커밋한다.
3. VSCode Git에서 `Rolling back fixes`가 보이면 커밋을 중단하고, 터미널 기준으로 1→2 순서를 다시 수행한다.
4. 필요하면 `git diff --staged -- <파일>` 와 `git diff -- <파일>` 를 둘 다 확인해 staged/unstaged 차이를 먼저 없앤다.

**관련 파일**: `infra/remote-proxy/patch_lis_nginx_remote.py`, `.pre-commit-config.yaml`, `Makefile`, `docs/rules/error_analysis.md`

---

### [2026-03-27] pre-commit mypy `redundant-cast` (`agent.py:43`)로 커밋 차단

**오류 유형**: 정적 분석 / 타입 힌트 과잉 (mypy `redundant-cast`)

**발생 상황**: VSCode Git 커밋에서 `ruff`, `ruff-format`은 통과했지만 mypy가 `idr_analytics/app/api/v1/endpoints/agent.py:43`에 대해 `Redundant cast to "str"`를 보고해 커밋이 실패함.

**근본 원인**: `strftime("%Y-%m")`는 이미 `str` 반환인데, 안전을 위해 추가한 `cast(str, ...)`가 오히려 불필요한 캐스트로 판정됨.

**재발 방지 규칙**:
1. mypy strict 환경에서 표준 라이브러리 함수(`strftime`, `str()`, `json.dumps` 등)가 이미 구체 타입을 보장하면 `cast`를 추가하지 않는다.
2. `cast`는 외부 라이브러리 스텁이 `Any`일 때처럼 **타입 정보가 실제로 부족한 지점에만** 제한적으로 사용한다.
3. 커밋 전 `PYTHONPATH=idr_analytics poetry run mypy idr_analytics/app/api/v1/endpoints/agent.py --strict`로 단일 파일 확인 후 전체 훅을 돌린다.

**관련 파일**: `idr_analytics/app/api/v1/endpoints/agent.py`, `docs/rules/error_analysis.md`

---

### [2026-03-27] MCP(ga-server SSH)로 «프록시 확인» 지시를 원격 배포·파일 반영으로 확대 해석

**오류 유형**: 프로세스 위반 / 범위 과대 적용 (운영·인프라)

**발생 상황**: 사용자가 `lis.*` 공인 URL의 `/ide/…` 404를 두고 **MCP(`user-ga-server-ssh`)로 ga-server에 접속해 프록시(nginx) 설정만 확인**하라고 했음에도, AI가 동일 MCP로 원격에 정적 파일 전송·`docker-compose` 수정·컨테이너 재기동 등 **배포 행위**까지 시도함. 사용자는 이를 중단하고 «내부에서 원인 검토·저장소 수정만» 요구함.

**근본 원인**: (1) «확인」을 «문제 해결까지 한 번에」로 확대 해석함. (2) `/ide` JSON 404의 원인을 nginx만으로 가정하고, **업스트림 `idr-fastapi`의 `demo/ide` 유무**와 **사용자가 배포를 명시했는지**를 구분하지 않음.

**재발 방지 규칙**:
1. **`user-ga-server-ssh` / ga-server MCP**: 사용자가 **명시적으로** «원격에 반영」「배포」「컨테이너 재시작」「서버에 파일 작성」을 요청하지 않는 한, **읽기 전용 조회**로만 사용한다. 예: `docker exec ga-nginx`로 `grep`/`sed -n` 등 **설정 내용 확인**, `curl`/`wget`으로 업스트림 응답 코드 확인. **원격에 바이너리·HTML 업로드, compose 실제 변경, `docker compose up` 등은 하지 않는다.**
2. **`https://lis.…/ide/…` 가 `{"detail":"Not Found"}`**: **먼저** nginx에서 `location /ide/` + `proxy_pass …/ide/` 가 **`/api/v1/` 과 동일 업스트림**인지 확인. **팀 운영 표준(`lis_public_url_path_map.md` §0)** 은 **로컬 우회**이므로 그다음은 **로컬** `demo/ide`·uvicorn·터널 가동 여부다. **`idr-fastapi` 이미지·볼륨**은 **예외(패턴 A) 합의**가 있을 때만 1차 처방으로 둔다.
3. 동일 증상을 **nginx 스니펫만 반복 수정**하지 않는다. 원인 분기는 **`docs/rules/project_context.md` §ga-server·MCP·`/ide` 404** 및 **`infra/deploy/public-edge/README.md`** 「`/ide/…` 가 … 일 때」를 따른다.

**관련 파일**: `docs/rules/project_context.md`, `infra/remote-proxy/README.md`, `infra/deploy/public-edge/README.md`, `idr_analytics/app/main.py`, `Dockerfile`, `infra/deploy/public-edge/docker-compose.idr-stack.yml`

---

### [2026-03-28] (재발) MCP를 «ga-server Docker nginx 확인」이 아닌 idr-fastapi·호스트 배포까지 사용

**오류 유형**: 프로세스 위반 / 도구 범위 초과 (사용자가 문서에 명시한 MCP 용도 위반)

**발생 상황**: 사용자는 **MCP(`user-ga-server-ssh`)로 ga-server에 접근하는 목적**을 **서버 안 Docker로 떠 있는 nginx(예: `ga-nginx`) 설정을 확인하고, 필요 시 그 부분만 수정**하도록 문서에 제한해 두었음에도, AI가 동일 MCP로 **`idr-fastapi` 컨테이너 내부 조회·404 원인 귀결 후 파일 업로드·compose 변경 시도** 등 **엣지 nginx 밖** 작업을 반복함.

**근본 원인**: (1) «404 해결」에 MCP 한 가지 수단만 연결해 **모든 원격 조작**을 허용한 것처럼 행동함. (2) **허용 범위 = `ga-nginx` 컨테이너 내 설정**이라는 **단일 경계**를 문서·스킬에 충분히 굵게 쓰지 않음.

**재발 방지 규칙**:
1. **`user-ga-server-ssh`의 유일한 기본 허용 범위**: **`docker exec ga-nginx` (또는 프로젝트가 쓰는 엣지 nginx 컨테이너명)** 으로 **설정 파일 읽기**·사용자가 **명시적으로** 「nginx 설정 수정」이라고 한 경우에 한해 **그 컨테이너에 실린 conf** 편집·`nginx -t`/reload 안내.
2. **동일 MCP로 금지**: `idr-fastapi`·DB·Redis·Dify 등 **다른 컨테이너** 조작, ga-server **호스트 파일 트리에 정적 자산·스크립트 쓰기**, **`docker compose`·이미지 빌드·청크 전송 배포**. 이들은 **앱/배포 담당**이 리포·절차로 처리.
3. `/ide` JSON 404는 nginx가 맞게 붙였는지 **1번 범위로만** 확인한 뒤, 남는 원인은 **업스트림 FastAPI·리포**로 넘긴다. **nginx만 반복 수정하거나 MCP로 앱 서버를 고치려 하지 않는다.**

**관련 파일**: `docs/rules/project_context.md` 「ga-server·공인 URL·MCP」, `.cursor/skills/idr-session-workflow/SKILL.md`, `infra/remote-proxy/README.md`

---

### [2026-03-28] ga-nginx: 호스트 `go-almond.swagger.conf` 와 컨테이너 `default.conf` md5 불일치

**오류 유형**: 운영 점검 착오 — 편집한 호스트 파일과 엣지가 실제로 읽는 내용이 다름

**발생 상황**: 호스트 경로의 설정은 `proxy_pass http://172.18.0.1:8000/…` 인데, `docker exec ga-nginx` 로 본 `/etc/nginx/conf.d/default.conf` 는 `http://idr-fastapi:8000/…` 등 **다른 스니펫**. `md5sum` 이 **서로 다름**.

**근본 원인**: bind 마운트가 **Inspect 상으로는** 올바른데, 런타임에 컨테이너가 **오래된 내용**을 보고 있거나(재시작 전 레이어·환경 이슈), 변경 후 **호스트만 보고** 컨테이너 내부를 확인하지 않음.

**재발 방지 규칙**:
1. 엣지 반영 여부는 **`md5sum` 호스트 원본 = `docker exec ga-nginx md5sum …/default.conf`** 로 확인한다. 불일치면 **`docker restart ga-nginx`** 후 재비교, 이어서 `nginx -t`·reload.
2. 공인 동작이 conf 와 안 맞으면 **컨테이너 안 파일**을 `grep`/`sed -n` 으로 본다(호스트 파일만 신뢰하지 않음).

**관련**: `docs/CURRENT_WORK_SESSION.md` — 공인 `/ide` 「검증 스냅샷」.

---

### [2026-03-28] 공인 `/ide/…` 를 «데모와 다른 제품 방향»으로 오해

**오류 유형**: 아키텍처 오해 / 문서·작업 방향 분산

**발생 상황**: `https://lis…/` 에 데모·`/apps` 에 Dify 는 이미 두었는데, **`/ide/docs/rules/` 교육생 가이드가 별도 호스트·별도 배포 축**처럼 설명되거나, “다른 방향으로 작업한 것 같다”는 피드백이 나옴.

**근본 원인**: (1) **단일 호스트·경로(prefix) 분리**라는 운영 의도가 한 문서에 고정되지 않아, `/ide` 만 다른 서비스처럼 취급됨. (2) 루트 `/` 가 nginx 정적 `demo/` 이고 `/ide` 는 FastAPI `StaticFiles` 인 점이 생략되어 **“데모 됐 = /ide 됐”** 으로 착각.

**재발 방지 규칙**:
1. 공인 URL 역할 표는 **`docs/plans/lis_public_url_path_map.md`** 를 정본으로 하고, `project_context.md`·`ppt_aux_instructor_build_guide.md`·`student_rules_download_lis_plan.md`·`infra/remote-proxy/*` 는 이와 **모순되지 않게** 유지한다.
2. `/ide` 404 시 원인은 (a) 엣지 `location /ide/` (b) **업스트림 FastAPI의 `demo/ide` 실존** 순으로 본다 — “규칙만 다른 URL 체계로 새로 뚫는다”가 아니다.

**관련 파일**: `docs/plans/lis_public_url_path_map.md`, `docs/rules/project_context.md`, `infra/remote-proxy/ga-server-append-lis.qk54r71z.conf.snippet`

---

### [2026-03-28] 문서에 «패턴 A/B 동등»·`idr-fastapi` 기본값 → 운영은 로컬 우회(§0)인데 AI가 반대로 처리

**오류 유형**: 문서 모순 / 운영 원칙 미명시로 인한 반복 오판

**발생 상황**: 팀은 공인 `lis.*` FastAPI 를 **오직 로컬 터널 우회**로 운영한다고 일관되게 말했으나, 문서·스크립트에 **패턴 A(ga-server `idr-fastapi`)** 와 B 가 나란히 있거나 **A가 기본값**이면, AI·담당자가 **공인 404 → 서버에서 이미지 재빌드·compose·호스트에 `demo/ide` 심기**를 먼저 시도함.

**근본 원인**: (1) **운영 표준을 §0처럼 한 줄로 고정하지 않음**. (2) `public-edge`·패치 스크립트 등이 **기술적으로 가능한 배포**를 **팀 표준**과 동일시함.

**재발 방지 규칙**:
1. **`docs/plans/lis_public_url_path_map.md` §0** 를 정본으로 — 공인 FastAPI 는 **로컬 우회만**이 팀 표준, 패턴 A 는 **예외·별도 합의**.
2. `patch_lis_nginx_remote.py` 기본 `FASTAPI_UPSTREAM`·`infra/remote-proxy/README.md`·`project_context.md`·`.cursorrules`·`SKILL.md` 는 §0 와 **모순되지 않게** 유지한다.
3. AI는 공인 장애 시 **`idr-fastapi` 재빌드를 운영 표준 처방으로 제시하지 않는다.**

**관련 파일**: `docs/plans/lis_public_url_path_map.md` §0, `infra/remote-proxy/patch_lis_nginx_remote.py`, `infra/deploy/public-edge/README.md`, `docs/rules/project_context.md`, `.cursor/skills/idr-session-workflow/SKILL.md`

---

### [2026-03-27] 세션 상세 계획을 `docs/plans/<신규>.md`에 작성 — `CURRENT` 규칙 위반

**오류 유형**: 프로세스 위반 / 문서 체계 오판

**발생 상황**: 사용자가 「동일 배포에서 교육생 가이드」 등 **상세 계획** 작성을 요청했을 때, `docs/rules/workflow_gates.md`·`.cursorrules`에 정한 **`docs/CURRENT_WORK_SESSION.md` 단일 기록**을 따르지 않고 `docs/plans/lis_trainee_ai_guide_same_host_plan.md` 같은 **새 plans 파일**을 만들었다.

**근본 원인**: (1) `docs/plans/`가 WBS·참조용인지 **세션 Gate A 상세용인지** 구분을 하지 않음. (2) `CURRENT`에 이미 동일 주제 절이 있어도 **중복·분산**이 낫다고 잘못 판단함.

**재발 방지 규칙**:
1. Gate A·C **상세**(실행 순서, Phase 표, DoD, 체크리스트)는 **`docs/CURRENT_WORK_SESSION.md`에만** 쓴다.
2. **`docs/plans/`에 세션 전용 상세 계획 마크다운을 신설하지 않는다** — 허용되는 것은 `plan.md`, 경로 정본, 강의·패키지 등 **배경·참조** 문서뿐이다.
3. 기존 내용 보강은 **`CURRENT` 해당 절 편집** 또는 Gate E 후 `WORK_HISTORY.md`로 이전한다.
4. 규정 강화 단일 원본: `workflow_gates.md` Gate A·Gate C·AI 금지, `SKILL.md` 「상세 계획 문서 위치」, `.cursorrules` `CURRENT` 항목.

**관련 파일**: `docs/CURRENT_WORK_SESSION.md`, `docs/rules/workflow_gates.md`, `.cursor/skills/idr-session-workflow/SKILL.md`, `.cursorrules`, `docs/rules/project_context.md`
