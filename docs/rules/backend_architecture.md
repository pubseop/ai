# 백엔드 아키텍처 규칙

> 이 규칙은 `labeling_vscode_claude` 프로젝트에서 검증된 패턴을 IDR 분석 시스템에 적용한 것이다.
> 기반 패턴 원본: `labeling_vscode_claude/docs/SDD.md`, `LESSONS_LEARNED.md`, `PLAN_01_project_scaffolding.md`

---

## 1. 비동기(async/await) API 작성 규칙

### 기본 원칙: 모든 엔드포인트와 서비스 메서드는 async

```python
# 올바른 패턴
@router.post("/scm/forecast")
async def create_forecast(
    body: ForecastRequest,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
) -> ForecastResponse:
    return await scm_service.forecast(body, db)

# 잘못된 패턴 — def 사용 금지 (FastAPI async 생태계 전제)
@router.post("/scm/forecast")
def create_forecast(...):  # WRONG
    ...
```

### FastAPI 엔드포인트 파일에서 TYPE_CHECKING 사용 금지 (LESSONS_LEARNED 교훈)

```python
# 잘못된 패턴 — endpoints/*.py 파일에 절대 사용 금지
from __future__ import annotations
if TYPE_CHECKING:
    from app.models.user import User  # Depends가 User를 런타임에 resolve 못함 → 422 에러

# 올바른 패턴 — 직접 import
from app.models.user import User  # endpoints/*.py에서는 항상 직접 import
```

> **이유**: FastAPI는 `Annotated[User, Depends(...)]` 파라미터에서 타입을 런타임에 평가한다.
> `TYPE_CHECKING` 블록 안의 타입은 런타임에 없으므로 Depends를 인식하지 못하고 query parameter로 fallback → 422 Unprocessable Entity.
> `from __future__ import annotations`는 `app/services/**`, `app/models/**`에서는 사용 가능하다.

### SQLAlchemy Async 세션 패턴

```python
# app/db/session.py — 반환 타입 명시 필수 (mypy strict)
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        yield session
```

### CPU-bound 동기 라이브러리와 `async def` 서비스 (Prophet, scikit-learn 등)

서비스 메서드는 **`async def`** 로 유지하되, **Prophet `fit`/`predict`**, **KMeans `fit`**, **대용량 Pandas 순회** 등 순수 CPU-bound 동기 호출을 `async def` 본문에서 **직접** 호출하면 이벤트 루프가 차단된다.

**권장 패턴** — 전용 스레드 풀에서 실행:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

_executor = ThreadPoolExecutor(max_workers=4, thread_name_prefix="analytics")

def _sync_prophet_forecast(df: pd.DataFrame, periods: int) -> pd.DataFrame:
    ...

async def forecast(self, df: pd.DataFrame, periods: int) -> pd.DataFrame:
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(
        _executor,
        lambda: _sync_prophet_forecast(df, periods),
    )
```

**대안**: 장시간 분석은 **ARQ 백그라운드 잡**으로만 실행하고, HTTP 핸들러는 job_id 반환·폴링만 담당한다 (`plan.md` Phase 5 ARQ 항목).

---

## 2. 하이브리드 라우팅 패턴 (핵심 아키텍처)

### labeling_vscode_claude에서 차용한 2-Tier 패턴

라벨링 프로젝트에서 "이미지 신뢰도 < 85% → VLM Tier 2 에스컬레이션" 패턴을 정형 데이터 분석에 재적용:

| 라벨링 프로젝트 (원형) | IDR 분석 시스템 (적용) | Spring Boot 비유 |
|----------------------|----------------------|-----------------|
| `ExtractionService` | `AnalysisRoutingService` | @Service AnalysisService |
| Surya OCR (Tier 1) | Pandas Analytics (Tier 1) | @Component StatisticsEngine |
| Ollama VLM (Tier 2) | Dify + LLM (Tier 2) | @Component LLMGateway |
| `confidence_threshold (0.85)` | `AI_ESCALATION_THRESHOLD (70)` | @Value("${analysis.threshold}") |

### ComplexityScorer — 라우팅 결정 로직

```python
# app/core/routing.py

class ComplexityScorer:
    WEIGHTS = {
        QueryType.AGGREGATION:      10,
        QueryType.FORECAST:         30,
        QueryType.CLUSTER:          35,
        QueryType.TREND:            25,
        QueryType.NATURAL_LANGUAGE: 80,  # 즉시 Tier 2
        QueryType.ANOMALY_EXPLAIN:  75,
    }

    def score(self, request: AnalysisRequest) -> ComplexityScore:
        base = self.WEIGHTS[request.query_type]
        # ... size_penalty, cross_bonus 계산
        total = base + size_penalty + cross_bonus
        return ComplexityScore(
            score=total,
            route=Route.AI if total >= settings.AI_ESCALATION_THRESHOLD else Route.PANDAS
        )
```

### 라우팅 결정 매트릭스

| 쿼리 유형 | Pandas Tier 1 | Dify Tier 2 |
|----------|:---:|:---:|
| 단순 집계 (sum, count, mean) | ✅ | ✗ |
| 시계열 예측 (Prophet/ARIMA) | ✅ | ✗ |
| K-Means 클러스터링 | ✅ | ✗ |
| 자연어 쿼리 (한국어) | ✗ | ✅ |
| 이상치 원인 설명 | ✗ | ✅ |
| 교차 도메인 복합 분석 | ✗ | ✅ |
| 인사이트 문장 생성 | ✗ | ✅ |

---

## 3. 의존성 주입(DI) 패턴

### labeling_vscode_claude의 Depends 패턴을 그대로 적용

```python
# app/core/dependencies.py
# Spring Boot @Autowired / @PreAuthorize 에 해당

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
) -> User:
    """Access Token 검증 → User 객체 반환. Spring: UserDetailsService.loadUserByUsername()"""
    payload = verify_token(token)
    user = await user_crud.get(db, id=payload.sub)
    if user is None:
        raise HTTPException(status_code=401)
    return user

async def require_admin(current_user: User = Depends(get_current_user)) -> User:
    """Spring: @PreAuthorize("hasRole('ADMIN')")"""
    if current_user.role != "admin":
        raise HTTPException(status_code=403)
    return current_user
```

### 엔드포인트에서의 서비스 주입

```python
# 서비스 인스턴스를 모듈 레벨에서 생성하고 Depends(lambda: ...)로 주입
# labeling_vscode_claude/labeling.py 패턴 그대로 적용

from app.services.analytics.scm_service import scm_service  # 모듈 레벨 인스턴스

@router.post("/scm/forecast")
async def create_forecast(
    body: ForecastRequest,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
    svc: SCMService = Depends(lambda: scm_service),  # DI
) -> ForecastResponse:
    return await svc.forecast(body, db)
```

---

## 4. Pandas 서비스 계층 분리 원칙

### 계층별 책임 분리

```
[엔드포인트 계층]  app/api/v1/endpoints/
    - HTTP 요청/응답 처리
    - Pydantic 스키마 변환
    - DI(Depends)를 통한 서비스 호출만

[라우팅 계층]  app/services/analytics/routing_service.py
    - ComplexityScorer로 Tier 1/2 결정
    - SCMService / CRMService / BIService / AgentService로 위임

[데이터 전처리 계층]  app/services/data/preprocessing_service.py
    - CSV → 검증된 DataFrame
    - 결측치 forward-fill
    - DatetimeIndex 생성
    - Lag / rolling window 피처 엔지니어링
    - **Pandas 연산 로직은 반드시 이 계층에서만 수행**

[분석 계층]  app/services/analytics/
    - scm_service.py: Prophet / ARIMA 실행
    - crm_service.py: K-Means + RFM 계산
    - bi_service.py: GroupBy Pivot + 집계
    - 이 계층은 이미 정제된 DataFrame을 받아서 처리

[AI 오케스트레이션 계층]  app/services/ai/agent_service.py
    - Dify Workflow API 호출 (httpx)
    - Pandas 연산 절대 금지 — 데이터는 FastAPI 엔드포인트에서 전처리 후 전달
```

### Pandas 코드 작성 원칙

```python
# app/services/data/preprocessing_service.py

class PreprocessingService:
    """CSV → 분석 가능한 DataFrame 변환. Spring: @Component DataTransformService"""

    def build_time_index(self, df: pd.DataFrame, date_col: str) -> pd.DataFrame:
        """DatetimeIndex 생성 + 결측치 forward-fill."""
        df = df.copy()  # 불변성 원칙 — 원본 변경 금지
        df[date_col] = pd.to_datetime(df[date_col])
        df = df.set_index(date_col).sort_index()
        df = df.ffill()  # forward-fill
        return df  # 새 DataFrame 반환

    # 규칙: df를 in-place로 수정하지 말 것 — df.copy() 후 반환
```

### HTTP/API 에러 응답 표준 (클라이언트·Dify 공통)

FastAPI 기본 `HTTPException(detail=...)`는 JSON으로 `{"detail": ...}` 형태가 된다. **Dify HTTP Request Node**가 파싱하기 쉽게 하려면 다음을 권장한다.

1. **422/401/403**: FastAPI 기본 유지 가능 (`detail` 문자열 또는 검증 오류 배열).
2. **404/409 등 도메인 오류**: `detail`에 **문자열**만 두지 말고, 가능하면 **일관된 dict**를 둔다.
   ```python
   raise HTTPException(
       status_code=404,
       detail={"code": "DATASET_NOT_FOUND", "message": "dataset not found", "dataset_id": str(dataset_id)},
   )
   ```
3. **5xx**: 내부 메시지는 로그에만 남기고, 응답 `detail`은 짧은 공개 메시지로 제한한다.

프로젝트 전역으로 커스텀 예외 클래스를 도입할 경우, **단일 `exception_handler`** 에서 위 형태로 직렬화하는 것이 유지보수에 유리하다. 상세는 Phase 5 구현 시 `main.py`에 합의 후 반영한다.

---

## 5. 코드 품질 게이트 (pre-commit 통과 필수)

### 하네스 도구 (labeling 프로젝트와 동일한 설정)

| 도구 | 역할 | Spring Boot 대응 |
|------|------|-----------------|
| **Ruff** | 린트 + 자동 포매팅 (line-length=120) | Checkstyle + Google Java Format |
| **Mypy (strict)** | 정적 타입 강제 | javac 타입 컴파일 + SonarQube |
| **pre-commit** | 커밋 직전 자동 실행 | Git Hooks |

### ruff 핵심 규칙 (labeling 프로젝트 동일 설정)

- `line-length = 120` — 한국어 주석 포함 시 88자 초과 빈번
- `app/models/**`: `TCH003` 무시 — SQLAlchemy ORM에서 uuid/datetime 런타임 참조 필요
- `app/schemas/**`: `TCH003` 무시 — Pydantic 런타임 참조 필요
- `alembic/**`: `E501`, `TCH` 무시 — SQL DDL 긴 줄 허용
- Alembic downgrade에서 f-string SQL 절대 금지 → `sa.table().delete().where()` 패턴 사용

### mypy 주의 사항 (LESSONS_LEARNED 교훈)

```python
# SQLAlchemy WHERE 절에서 Python bool 리터럴 사용 금지
# 잘못된 패턴
base_filter = True  # mypy: bool != ColumnElement[bool]

# 올바른 패턴
from sqlalchemy import true
base_filter = true()  # ColumnElement[bool]
```

### 커밋 전 체크리스트

- [ ] `git add <파일>` 먼저 실행 (staged 동기화 후 커밋)
- [ ] pre-commit 훅 통과 확인
- [ ] 미사용 변수 할당 없음 (ruff F841)
- [ ] TYPE_CHECKING 블록 위치 올바름 (endpoints/*.py에서는 사용 금지)

---

## 6. 테스트 전략

### pytest 실행 (컨테이너 빌드 불가 — 호스트 직접 실행)

```bash
PYTHONPATH=. pytest tests/
PYTHONPATH=. pytest tests/unit/
PYTHONPATH=. pytest tests/ --cov=app --cov-report=term-missing
```

### 테스트 격리 원칙 (labeling 프로젝트 교훈 반영)

```python
# tests/unit/conftest.py — 파일 최상단에서 env var 사전 설정 필수
# module-level에서 settings.X를 참조하는 코드의 pytest collection 오류 방지
import os
os.environ.setdefault("DATABASE_URL", "postgresql+asyncpg://test:test@localhost:15432/idr_test")
os.environ.setdefault("REDIS_URL", "redis://localhost:6379/1")
os.environ.setdefault("SECRET_KEY", "test-secret-key")
```

> **이유**: `app/db/session.py`가 module-level에서 `settings.DATABASE_URL`을 즉시 참조한다.
> pytest collection 단계에서 test 모듈 import 시 env var가 없으면 `pydantic ValidationError` 발생.
> `autouse` monkeypatch fixture는 collection 이후 실행되므로 collection 오류를 막지 못한다.

### Dify AgentService Mock 패턴

```python
# Dify Workflow API 호출은 반드시 Mock 처리
with patch("app.services.ai.agent_service.httpx.AsyncClient") as mock_client:
    mock_response = MagicMock()
    mock_response.json.return_value = {"data": {"outputs": {"result": "분석 완료"}}}
    mock_client.return_value.__aenter__.return_value.post.return_value = mock_response
    ...
```
