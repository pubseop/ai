# Dify 연동 규칙 — FastAPI ↔ Local Self-hosted Dify

> **핵심 원칙**: Dify는 데이터를 직접 처리하지 않는다.
> 복잡한 수치 계산은 반드시 FastAPI에 위임하고, 정제된 JSON만 받아서 LLM에 전달한다.
> Spring Boot 비유: Dify = Camunda (BPM 워크플로 엔진) + LLM 호출 내장

---

## 1. 역할 분리 원칙

| 계층 | 담당 | 처리 내용 |
|------|------|----------|
| **데이터 처리 계층** | FastAPI + Pandas | CSV 파싱, 결측치 처리, Prophet/K-Means 실행, JSON 직렬화 |
| **AI 오케스트레이션 계층** | Self-hosted Dify | HTTP Request → LLM Node 체인, 멀티스텝 프롬프팅, 워크플로 시각화 |
| **LLM 실행 계층** | Claude API / Ollama | 자연어 이해, 인사이트 생성, 한국어 요약 |

---

## 2. REST API 응답 JSON 포맷 표준

### 원칙: Dify LLM Node의 컨텍스트 창을 낭비하지 않는다

Dify에서 FastAPI를 호출할 때 응답 크기가 크면 LLM 컨텍스트 창이 낭비된다.
따라서 **모든 분석 엔드포인트는 `compact=true` 쿼리 파라미터를 지원해야 한다.**

```python
# 모든 분석 엔드포인트 공통 패턴
# app/api/v1/endpoints/crm.py

@router.get("/crm/churn-risk")
async def get_churn_risk(
    dataset_id: UUID,
    top_n: int = 20,
    compact: bool = False,       # Dify HTTP Request Node → ?compact=true 로 호출
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user),
) -> dict:
    result = await crm_service.compute_churn_risk(dataset_id, db)

    if compact:
        # Dify용: 핵심 필드만 반환 — LLM 컨텍스트 절약
        return {
            "high_risk_count": len(result.high_risk),
            "top_customers": [
                {"code": c.code, "name": c.name, "risk_score": c.score}
                for c in result.high_risk[:top_n]
            ],
            "summary": result.summary_stats,
        }

    return result  # 일반 클라이언트용: 전체 응답
```

### compact=true 응답 설계 기준

| 항목 | 기준 |
|------|------|
| 최대 리스트 항목 수 | 기본 20개 (`top_n` 파라미터로 조정 가능) |
| 포함 필드 | 식별자(code/id), 핵심 지표 1~3개, 요약 텍스트 1개 |
| 제외 필드 | 타임스탬프, 메타데이터, 중간 계산 결과, 원본 배열 전체 |
| 응답 크기 목표 | 4KB 이하 (LLM 컨텍스트 1,000 토큰 이내) |

### 도메인별 compact 응답 형식

**SCM (수요 예측):**
```json
{
  "forecast_period_days": 30,
  "high_demand_items": [
    {"test_code": "BRCA1", "predicted_qty": 142, "trend": "increasing"}
  ],
  "restock_alerts": 3,
  "summary": "BRCA1 수요 14% 증가 예상. 3개 품목 재고 보충 필요."
}
```

**CRM (고객 분석):**
```json
{
  "high_risk_count": 12,
  "top_customers": [
    {"code": "C001", "name": "서울대병원", "risk_score": 0.87}
  ],
  "cluster_count": 4,
  "summary": "12개 거래처 이탈 위험. 서울 권역 집중 관리 필요."
}
```

**BI (트렌드 분석):**
```json
{
  "period": "2026-01",
  "top_regions": ["서울", "부산", "대구"],
  "trending_tests": ["BRCA1", "HPV"],
  "heatmap_highlights": [
    {"region": "서울", "test": "BRCA1", "growth_rate": 0.23}
  ],
  "summary": "서울 BRCA1 검사 23% 증가. 계절성 패턴 확인됨."
}
```

---

## 3. Dify HTTP Request Node 연동 가이드

### 3.0 관리자 계정·로그인 완료 후 체크리스트 (순서 고정)

워크플로에서 FastAPI를 부를 때는 **동기 JSON 응답**이 나오는 엔드포인트를 쓴다.  
`/crm/cluster`는 **202 + job_id**(ARQ)라서 Dify 체인에 바로 넣기 어렵다 → **`/crm/churn-risk`(GET)** 를 사용한다.

| 순서 | 작업 | 비고 |
|:----:|------|------|
| 1 | **Settings → Model Provider** 에서 Anthropic(또는 Ollama) 설정 | Dify가 LLM 노드를 실행하려면 필수 |
| 2 | **Studio → Create app → Workflow** (예: 이름 `IDR CRM 이탈 분석`) | Chat이 아닌 Workflow 타입 |
| 3 | Workflow **시작 변수**: `user_query` (paragraph), `dataset_id` (short text) | `AgentService`가 `inputs`로 동일 키 전달 |
| 4 | Studio **API Access → API Key** 발급 → 프로젝트 `.env`의 `DIFY_API_KEY` | `app-...` 형식 |
| 5 | 앱 **API URL** 또는 Studio에서 확인한 **Workflow / App id** → `.env`의 `DIFY_WORKFLOW_ID` | 버전별 UI 문구 상이; 복사한 UUID/문자열 그대로 |
| 6 | 아래 §3.2대로 HTTP Request 2개 + Aggregator + LLM + Answer 구성 | FastAPI는 호스트에서 띄운 경우 URL은 §3.1. **또는** 저장소 [`infra/dify/workflows/idr_crm_bi_tier2.yml`](../../infra/dify/workflows/idr_crm_bi_tier2.yml) DSL 가져오기 → [`infra/dify/workflows/README.md`](../../infra/dify/workflows/README.md) 에서 가져온 뒤 JWT·모델 확인 |
| 7 | **Publish** 후 Explore 또는 `/agent/query`(Tier2)로 스모크 | `docs/CURRENT_WORK_SESSION.md` Gate C |

### 3.1 FastAPI 베이스 URL (Dify 컨테이너 → 호스트)

| 실행 방식 | HTTP Request 의 URL 예 |
|-----------|-------------------------|
| FastAPI를 **호스트**에서 `uvicorn` (포트 8000) | `http://host.containers.internal:8000/api/v1/...` (Podman/Linux에서 동작하지 않으면 호스트 브리지 IP 사용) |
| FastAPI가 **idr-net** 상의 컨테이너 | `http://<서비스명>:8000/api/v1/...` |

모든 분석 호출에 **`compact=true`** 를 붙인다.

### 3.2 동기 호출 예시 (HTTP Request 노드)

**노드 A — 이탈 위험**

- 메서드: **GET**
- URL: `http://host.containers.internal:8000/api/v1/crm/churn-risk`
- Query: `dataset_id` = `{{#dataset_id#}}` (또는 해당 앱의 변수 치환 문법), `compact` = `true`, `top_n` = `10`
- Headers: `Authorization: Bearer <FastAPI JWT>` — 로컬 발급: 프로젝트 루트 `make dify-fastapi-jwt-bearer` (`.env`의 `IDR_LOGIN_USERNAME` / `IDR_LOGIN_PASSWORD`, 선택 `IDR_API_BASE_URL`). Dify **Secret**에 토큰만 저장 후 `Authorization: Bearer {{#env....#}}` 로 참조 가능(버전별 UI에 맞게 조정).
- 출력 변수명: `crm_result`

**노드 B — 지역 히트맵**

- 메서드: **GET**
- URL: `http://host.containers.internal:8000/api/v1/bi/regional-heatmap`
- Query: `dataset_id`, **`period`** (필수 — CSV의 기간 컬럼 값과 동일한 문자열, 예: `2026-01`), `compact` = `true`
- Headers: 노드 A와 동일 Bearer
- 출력 변수명: `bi_result`

> **JWT 발급**: `POST /api/v1/auth/login` 로 사용자명·비밀번호로 access_token을 받아 Dify Secret에 넣는다. 장기 토큰이 없다면 만료 시 갱신 필요.

### Dify 워크플로 전체 흐름 (CRM 이탈 분석 — 권장)

```
① Start Node
   - 입력 변수: user_query (string), dataset_id (string)
   - (선택) period — regional-heatmap 용; 없으면 상수 노드로 기본월 지정

② HTTP Request → GET /api/v1/crm/churn-risk?compact=true&top_n=10
   - 출력: crm_result

③ HTTP Request → GET /api/v1/bi/regional-heatmap?compact=true&period=...
   - 출력: bi_result

④ Variable Aggregator
   - 입력: crm_result + bi_result → combined_data

⑤ LLM Node
   System: "당신은 의료 검사 영업 분석 전문가입니다. 주어진 데이터를 분석하여 한국어로 인사이트를 제공하세요."
   User:   "데이터: {{combined_data}}\n질문: {{user_query}}"
   - 출력: llm_answer

⑥ Answer Node → {{llm_answer}}
```

**비고**: 배치 클러스터만 필요하면 `POST /api/v1/crm/cluster` + 폴링 `GET /api/v1/crm/cluster/{job_id}` 를 별도 워크플로로 구성한다. Body는 `{"dataset_id": "<uuid>", "n_clusters": 4}` (`ClusterRequest`).

---

## 4. FastAPI → Dify 프록시 패턴 (방식 A)

자연어 쿼리를 FastAPI `/agent/query` 엔드포인트로 받아 Dify Workflow API로 프록시하는 방식.

```python
# app/services/ai/agent_service.py

class AgentService:
    """Dify Workflow API 프록시 서비스. Spring: @Component LLMGateway"""

    def __init__(self) -> None:
        self._dify_base_url = settings.DIFY_API_BASE_URL  # http://localhost/v1
        self._dify_api_key = settings.DIFY_API_KEY         # app-xxxx...

    async def analyze(self, query: str, dataset_id: str) -> AgentResult:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self._dify_base_url}/workflows/run",
                headers={"Authorization": f"Bearer {self._dify_api_key}"},
                json={
                    "inputs": {
                        "user_query": query,
                        "dataset_id": dataset_id,
                    },
                    "response_mode": "blocking",
                },
                timeout=120.0,
            )
            response.raise_for_status()
            data = response.json()
            return AgentResult(
                answer=data["data"]["outputs"]["answer"],
                workflow_run_id=data["workflow_run_id"],
            )
```

> **방식 B (강의 권장)**: 프론트엔드 또는 Dify Chat UI에서 FastAPI를 직접 HTTP Request Node로 호출.
> 코드 작성 없이 Dify 시각적 워크플로만으로 완성 가능. 방식 A보다 학습 목적에 적합.

---

## 5. 인프라 구성 (Docker 네트워크 공유)

### 포트 구성 (충돌 방지)

| 서비스 | 포트 | 파일 / 비고 |
|--------|------|---------------|
| FastAPI (idr_analytics) | 8000 | 호스트 uvicorn 또는 prod compose |
| Dify Web UI + API Gateway | **호스트 8080→컨테이너 80** | `infra/dify/` — `.env` 의 `EXPOSE_NGINX_PORT` |
| PostgreSQL (idr 전용) | 5432 (내부) / **15432** (호스트) | `docker-compose.dev.yml` 등 |
| PostgreSQL (Dify 전용) | **15434** (호스트 도구용) | `infra/dify/docker-compose.idr.yml` 가 `db_postgres` 노출 |
| Redis (Dify 내장) | 컨테이너 내부만 | 공식 스택 `redis` 서비스 |
| Redis (idr) | 6379 | `docker-compose.dev.yml` |
| Ollama (온프레미스 LLM) | 11434 | 호스트 직접 실행 |

### 네트워크 공유 — `idr-net` 브리지

```yaml
# docker-compose.dev.yml 등 (FastAPI / idr 인프라)
networks:
  idr-net:
    name: idr-net
    driver: bridge

# infra/dify/docker-compose.idr.yml — api / worker / worker_beat 가 idr-net 에 추가 연결
networks:
  idr_net:
    name: idr-net
    external: true
```

> 네트워크 공유 이유: Dify의 HTTP Request Node에서 `http://fastapi-app:8000`으로 직접 접근하기 위함.
> 포트 포워딩(localhost:8000) 대신 Docker 내부 DNS로 통신.

### 환경 변수 (루트 `env.example` / 실사용 `.env` 필수 항목 — `.env*` 는 Git 무시)

```dotenv
# Dify 연동 설정 (방식 A — FastAPI 프록시 시, 로컬 nginx 포트 8080 기준)
DIFY_API_BASE_URL=http://localhost:8080/v1
DIFY_API_KEY=app-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
DIFY_WORKFLOW_ID=your-workflow-id-here
```

---

## 6. Dify 연동 시 FastAPI 응답 설계 체크리스트

코드를 작성하기 전 아래 항목을 확인한다:

- [ ] 엔드포인트에 `compact: bool = False` 파라미터가 있는가
- [ ] `compact=True` 응답이 4KB 이하인가
- [ ] `compact=True` 응답의 최상위 키 이름이 명확한가 (Dify Variable Aggregator에서 `{{crm_result.key}}`로 참조)
- [ ] 응답 JSON에 한국어 요약 문자열 필드(`summary`)가 있는가
- [ ] Dify HTTP Request Node에서 파싱할 배열의 길이가 `top_n`으로 제한되는가
- [ ] 에러 응답 시 Dify가 처리할 수 있는 명확한 JSON 에러 메시지를 반환하는가
