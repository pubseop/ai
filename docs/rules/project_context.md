# 프로젝트 컨텍스트 — IDR 시스템 데이터 분석 AI 백엔드

> 이 파일을 읽는 AI에게: 코드를 작성하기 전에 이 문서 전체를 읽어라.
> 세부 아키텍처 결정 근거는 `ref_files/IDR_Data_Analysis_SDD.md` (v1.1.0)에 있다.

---

## 프로젝트 정체성

| 항목 | 내용 |
|------|------|
| **서비스명** | IDR 시스템 데이터 분석 AI 에이전트 백엔드 (`idr_analytics`) |
| **목적** | IDR 시스템즈의 LIS(유전자 검사 의뢰 관리) 데이터를 AI로 분석하여 비즈니스 의사결정 자동화 |
| **고객사** | IDR 시스템즈 — 유전자 검사 의뢰, 수가, 청구수금, 매출 등 정형 데이터 보유 |
| **기반 아키텍처** | `labeling_vscode_claude` 프로젝트의 2-Tier 하이브리드 라우팅 패턴을 정형 데이터(CSV)에 재적용 |
| **개발 방법론** | 문서 주도 개발(SDD) — 스펙 확정 후 구현 |
| **컨텍스트 전달 대상** | Cursor AI (이 `.cursorrules` 체계) |

---

## 3가지 비즈니스 목표 (절대 잊지 마라)

| # | 도메인 | 목표 | AI 기법 | 엔드포인트 접두사 |
|---|--------|------|---------|-------------------|
| BG-1 | **SCM** | 시약·키트 수요 예측 및 재고 최적화 | Prophet / ARIMA 시계열 예측 | `/api/v1/scm/` |
| BG-2 | **CRM** | 거래처 이탈 징후 감지 및 잠재 고객 클러스터링 | K-Means / RFM 분석 | `/api/v1/crm/` |
| BG-3 | **BI** | 지역·시기별 질환 트렌드 분석 | GroupBy Pivot + LLM 요약 | `/api/v1/bi/` |

---

## 기술 스택 (Spring Boot 비유 포함)

| 계층 | 기술 | 버전 | Spring Boot 비유 |
|------|------|------|------------------|
| API Framework | **FastAPI** | Python 3.12+ | Spring MVC + @RestController |
| 데이터 처리 | **Pandas 2.x**, NumPy | — | Stream API + DTO |
| 시계열 예측 | **Prophet** (Meta), statsmodels ARIMA | — | Spring Batch Job |
| 클러스터링 | **scikit-learn** (K-Means, DBSCAN) | — | MLflow Pipeline |
| AI 오케스트레이션 | **Self-hosted Dify** (Docker Compose 로컬 설치) | — | Camunda Workflow Engine |
| LLM 클라이언트 | **Claude API** (anthropic), Ollama (온프레미스 옵션) | — | OpenFeign HTTP Client |
| ORM | **SQLAlchemy 2.0** (Async) | — | JPA + Hibernate |
| 마이그레이션 | **Alembic** | — | Flyway |
| 비동기 워커 | **ARQ** (Redis-backed) | — | @Async + @EnableAsync |
| 캐시 | **Redis** | — | Spring Cache (@Cacheable) |
| DB | **PostgreSQL 15+** | — | PostgreSQL |
| HTTP 클라이언트 | **httpx** | ^0.27 | RestTemplate / WebClient |
| 컨테이너 | **podman-compose** (Docker Compose 호환) | — | Docker Compose |

---

## 프로젝트 디렉토리 구조

```
idr_analytics/
├── app/
│   ├── main.py                         # FastAPI app + lifespan
│   ├── api/v1/
│   │   ├── api.py                      # 라우터 통합
│   │   └── endpoints/
│   │       ├── auth.py
│   │       ├── datasets.py             # CSV 업로드·프로파일링
│   │       ├── scm.py                  # BG-1: 수요 예측
│   │       ├── crm.py                  # BG-2: 고객 클러스터링
│   │       ├── bi.py                   # BG-3: 트렌드 분석
│   │       └── agent.py                # Dify 프록시 (자연어 쿼리)
│   ├── core/
│   │   ├── config.py                   # Pydantic Settings
│   │   ├── dependencies.py             # FastAPI Depends
│   │   └── routing.py                  # ComplexityScorer
│   ├── db/
│   │   ├── session.py
│   │   └── base.py
│   ├── models/                         # SQLAlchemy ORM
│   ├── schemas/                        # Pydantic DTO
│   ├── services/
│   │   ├── data/
│   │   │   ├── ingestion_service.py    # CSV → validated DataFrame
│   │   │   └── preprocessing_service.py
│   │   ├── analytics/
│   │   │   ├── routing_service.py      # 하이브리드 라우터
│   │   │   ├── scm_service.py          # Prophet / ARIMA
│   │   │   ├── crm_service.py          # K-Means + RFM
│   │   │   └── bi_service.py           # GroupBy + Pivot
│   │   └── ai/
│   │       └── agent_service.py        # Dify Workflow API 프록시
│   ├── crud/
│   └── workers/
│       └── arq_worker.py
├── alembic/
├── tests/
│   ├── unit/
│   └── integration/
├── env.example                      # Git 추적 템플릿 (.env* 는 전부 ignore)
├── pyproject.toml
├── docker-compose.yml                  # PostgreSQL + Redis (idr 전용)
└── infra/dify/                        # Self-hosted Dify 1.13+ (vendor + docker-compose.idr.yml)
```

---

## 실행 환경 (고정 사항 — 매 세션 재확인 금지)

| 항목 | 값 | 비고 |
|------|----|----|
| **OS** | RHEL 8 (kernel 4.18.0-553.56.1.el8_10) | 구형 커널 |
| **컨테이너 런타임** | rootless podman (Docker CLI 호환) | `docker` 명령 = podman 에뮬레이션 |
| **컨테이너 관리** | `podman-compose` | docker-compose 호환 |
| **SELinux** | Enabled | `podman build` 중 apt-get 단계에서 메모리 보호 충돌 |
| **Python (호스트)** | 3.13.2 (`/home/lukus/miniconda3/bin/python`) | 프로젝트 타겟은 3.12 |
| **pytest 실행** | 호스트 miniconda에서 `PYTHONPATH=idr_analytics poetry run pytest idr_analytics/tests/` | 컨테이너 빌드 불가로 호스트 직접 실행 |

> **컨테이너 빌드 실패 원인**: RHEL 8 + rootless podman + SELinux → glibc RELRO 메모리 보호 충돌.
> ruff, mypy, pre-commit, pytest는 호스트 miniconda에서 직접 실행한다. 이는 Docker First 원칙의 예외가 아닌, 개발 워크플로 도구이기 때문이다.

---

## ga-server·공인 URL(`lis.*`)·MCP — 범위 (반복 오류 방지)

**맥락**: 공인 `https://lis.qk54r71z.freeddns.org/` 등은 엣지(예: **`ga-nginx` Docker 컨테이너** 안의 nginx)가 **로컬/사설망에서 뜬 FastAPI·Dify**로 넘기는 **역프록시**일 뿐이다.

### 공인 `lis.*` FastAPI — 운영 표준은 **로컬 우회만**

**단일 원본**: `docs/plans/lis_public_url_path_map.md` **§0**.

- 공인 **`/api/v1/`·`/health`·`/docs`·`/openapi.json`·`/ide/`** 는 **터널·사설망으로 노출된 로컬 uvicorn** 한 업스트림을 본다. **`idr-fastapi` 를 공인 기본 업스트림으로 두는 것은 팀 운영 표준이 아니다** (예외는 별도 합의·문서).
- ga-nginx Docker 에서 그 업스트림 주소는 **`docker exec ga-nginx ip route show default`** 의 게이트웨이 + 터널 포트(예: **172.18.0.1:8000** — `ga-api-platform` 브리지; docker0 만 쓰는 호스트는 **172.17.0.1** 일 수 있음).
- 장애 시 **먼저** nginx `proxy_pass` 일치·로컬 `demo/ide`·터널. **ga-server `idr-fastapi` 이미지 재빌드를 표준 처방으로 제시하지 않는다.**

### 공인 단일 호스트 — 경로 역할(정본)

**의도된 사용자 경험**(서브도메인 분리가 아니라 **동일 호스트·경로만 구분**):

| 경로 | 역할 |
|------|------|
| `/` | IDR Analytics **강의 데모** (브라우저 기본 진입 — 구성에 따라 nginx 정적 `demo/` 또는 동일 호스트의 다른 서빙) |
| `/apps` 등 | **Dify** (Studio·앱) |
| `/ide/…` | **교육생 규칙 가이드 HTML·ZIP** — **`/api/v1/` 과 같은 FastAPI 업스트림**으로 `proxy_pass` 되어야 함 |

표·오해 시 정리·“잘못된 방향” 제거는 **`docs/plans/lis_public_url_path_map.md`** 가 단일 정본이다. “데모(`/`)만 되면 `/ide`도 자동”은 **성립하지 않을 수 있음**(루트 정적 vs `/ide`는 FastAPI `StaticFiles` 마운트).

### `user-ga-server-ssh`(MCP) 로 **할 수 있는 것** (최우선·좁게 해석)

| 허용 | 설명 |
|------|------|
| **ga-nginx(또는 동일 역할의 엣지 nginx 컨테이너) 안 설정만** | `docker exec ga-nginx …` 로 **마운트된 `conf.d`·`nginx.conf` 등을 읽기**(cat/grep/sed -n). `location`·`proxy_pass`·`server_name` 이 의도와 맞는지 확인. |
| **사용자가 문구로 명시한 경우에 한해 같은 범위의 수정** | 예: 「ga-server Docker nginx 설정만 고쳐라」「`lis` 서버 블록의 `proxy_pass` 를 바꿔라」처럼 **엣지 nginx 설정 파일** 편집·`nginx -t`·reload 절차 **안내 또는 사용자가 승인한 편집**. |

### 동일 MCP로 **하면 안 되는 것** (같은 오류 반복 금지)

| 금지 | 이유 |
|------|------|
| **`idr-fastapi`·Postgres·Redis·Dify·기타 앱 컨테이너**에 `docker exec` 로 들어가 파일 쓰기·패키지 설치·재기동 | 앱·데이터 계층은 **엣지 nginx MCP 범위 밖**. |
| ga-server **호스트**의 `/home/…/lis_cursor` 등에 정적 파일·스크립트 업로드 | 배포·동기화는 담당자·CI·git, AI MCP 아님. |
| **`docker compose` / 이미지 빌드 / 바이너리·HTML 청크 전송**으로 404 «해결» 시도 | 증상이 앱 쪽이면 **리포·로컬 운영형·엣지 스택 담당 절차**로 처리. |
| nginx가 이미 올바른데 **반복적으로 스니펫만 수정** | `{"detail":"Not Found"}` 는 **업스트림 FastAPI** 응답일 수 있음. |

### 증상 분기 (짧게)

| **`/ide/…` → `{"detail":"Not Found"}`** | (1) **ga-nginx**에서 `/ide/` 가 **`/api/v1/` 과 동일 업스트림**인지 확인(`docs/plans/lis_public_url_path_map.md` §0·§2.1). (2) **§0 표준**이면 **로컬** `demo/ide`·uvicorn·터널. **`idr-fastapi` 컨테이너는 예외 합의 시에만** 1차 원인으로 본다. |

**AI 오판 사례**: `docs/rules/error_analysis.md` MCP·ga-server 관련 항목(2026-03-27 이후 누적).

**엣지 nginx 스니펫·예시**: `infra/remote-proxy/README.md`, `infra/deploy/public-edge/README.md`.

---

## 작업 프로토콜 (반드시 준수)

어떤 태스크를 받더라도 아래 Gate 순서를 지키고, **각 Gate 전환 전 개발자 승인**을 받는다. 절차 전문·금지 사항·`CURRENT` 표준 섹션은 **`docs/rules/workflow_gates.md`** 가 단일 원본이다. Cursor 에이전트용 요약은 **`.cursor/skills/idr-session-workflow/SKILL.md`** 를 참고한다. `docs/CURRENT_WORK_SESSION.md`의 워크플로 표와 동일하다.

**세션 상세 계획의 기록 위치**: Gate A·C의 실행 순서·체크리스트·DoD는 **`docs/CURRENT_WORK_SESSION.md`에만** 둔다. **`docs/plans/`에 세션 전용 상세 계획 파일을 신설하지 않는다** — `docs/plans/`는 WBS·경로 정본·참조 문서용이다.

| 단계 | Gate |
|------|------|
| `plan`/참조 확인 후 **구현 상세 계획** CURRENT 작성·사용자 승인 (코딩 전) | A |
| **구현** 후 구현 완료 요약·체크리스트·**사용자 승인** | B |
| **테스트 상세 계획** CURRENT·사용자 승인 (테스트 실행 전) | C |
| 테스트 실행·검증 결과 기록 | D |
| `WORK_HISTORY`·`plan.md` 체크·다음 세션 CURRENT | E |

**승인 범위**: Gate A에 대한 동의(예: 「진행해라」)는 **코딩·구현 완료(Gate B 작성)까지만**. 구현 직후 테스트(`make test` 등)는 **Gate C·D이며 별도 승인**이 필요하다. 상세는 `workflow_gates.md` §승인 범위 해석.

구현 시 관례: 소스에는 **클래스/함수 수준 Docstring만** (튜토리얼식 주석 금지). 설명은 Spring Boot / Kotlin / JPA / Flyway 비유를 사용한다. pytest 예시: `PYTHONPATH=idr_analytics poetry run pytest idr_analytics/tests/` (또는 `CURRENT`에 적힌 명령).

---

## Git · 추적 제외 및 비밀 관리 (`.gitignore`)

**목적**: 새 파일·새 도구·새 인프라 디렉터리가 생겨도 **실비밀·로컬 산출물이 실수로 커밋되지 않도록** 한다.

**단일 원본**: 저장소 루트 **`.gitignore`**. 여기에 패턴이 없는 유형의 로컬 전용 파일이 생기면, **같은 작업(브랜치/PR) 안에서 패턴을 추가**한 뒤 커밋한다. AI·인간 모두 `git add` 전에 이 절을 확인한다.

### 커밋하면 안 되는 것 (범주)

| 범주 | 예시 | 비고 |
|------|------|------|
| 환경 변수(실값) | **이름이 `.env`로 시작하는 모든 파일** (`.env`, `.env.dev`, `.env.example`, `vendor/.env.example` 등), `infra/dify/.env`, `.envrc` | Git 에는 **올리지 않음** |
| Compose 로컬 오버라이드 | `docker-compose.override.yml`, `docker-compose.local.yml`, `docker-compose.*.local.yml` | 팀원마다 다른 비밀·포트 |
| 비밀키·인증서 | `*.pem`, `id_rsa`, `*.key`, `*.p12`, `*.pfx`, `*.jks` | `*.key.example` 등 **예시 전용**은 예외 가능 |
| 클라우드·IaC 비밀 | `*credentials*.json`, `google-services.json`, `*.tfstate*`, 민감한 `*.tfvars` | `*.tfvars.example` 는 커밋 가능 |
| Dify·컨테이너 런타임 데이터 | `infra/dify/vendor/volumes/`, `infra/dify/vendor/nginx/ssl/*` (`.gitkeep` 제외) | DB·업로드·플러그인 캐시 |
| 언어·도구 캐시·커버리지 | `__pycache__/`, `.mypy_cache/`, `.pytest_cache/`, `.ruff_cache/`, `htmlcov/`, `coverage.xml`, `*.sqlite3` | CI/로컬 재현 산출물 |
| 내부 전용 문서 트리 | `ref_files/` | 레포 정책상 추적 제외 |

### 반드시 Git에 포함하는 것 (템플릿·예시만 — **`.env` 접두사 금지**)

- **`env.example`** (FastAPI 루트 템플릿), **`env.prod.example`**, **`infra/dify/env.vendor.example`** (Dify 템플릿): 변수 **이름·구조·플레이스홀더**만. API 키·운영 비밀번호는 넣지 않는다.
- **`.env*` 패턴으로 `.env`로 시작하는 파일은 예외 없이 무시**한다. 업스트림이 `.env.example` 이름만 쓰는 경우, 레포에는 **`env.*.example`** 등 점 없는 이름으로 복사해 둔다 (`make dify-env-bootstrap` 등).
- 새 템플릿을 추가할 때 **`git check-ignore -v <경로>`** 로 실수로 무시되는지 확인한다.

### 새 파일을 만들 때 절차 (체크리스트)

1. **`git status`** — 의도하지 않은 경로가 뜨지 않는지 본다.
2. 로컬 전용·비밀이면 **먼저 `.gitignore`에 패턴 추가** (또는 기존 패턴에 걸리는지 확인).
3. **`git check-ignore -v <경로>`** — 무시되는지 확인. 템플릿 파일명은 **`.env`로 시작하지 않게** 유지한다.
4. **`git add -p` 또는 경로 지정 `git add`** — `-A` 만 습관적으로 쓰지 않고, 특히 `infra/`·`.env*`·인증서 경로는 선택적으로 스테이징한다.

### 히스토리에 비밀이 올라간 경우

- 해당 파일을 추적에서 제거한 커밋을 만든 뒤, **키·비밀번호는 즉시 로테이션**한다.
- 공개 저장소·공유 원격이면 **`git filter-repo` 등으로 히스토리에서 객체 제거**를 검토하고, force push 전에 팀과 합의한다. (과거 커밋에 `POSTGRES_PASSWORD` 하드코딩이 있었던 사례는 이미 정리됨 — `docs/history/WORK_HISTORY.md`·관련 커밋 메시지 참고.)
