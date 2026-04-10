# 에러 처리와 로깅 원칙 — 학습 노트

| 항목 | 내용 |
|------|------|
| Date | 2026-04-10 |
| Context | Repository 레이어 도입 시 예외 설계 → 에러 처리 책임 분담 → 로깅 전략으로 확장 |
| 핵심 질문 | 어느 레이어가 무엇을 raise/catch하고, 어디서 로그를 찍는가? |

---

## 1. 호출 그래프 vs 추상도 — 레이어는 수직이 아니다

흔한 오해: "API → Tool → Worker → Engine → Repository → DB" 같은 수직 스택.

실제 구조 (Functional Core / Imperative Shell):

```
                ┌──────────────────────┐
                │  HTTP/API Layer      │
                └──────────┬───────────┘
                           │
                ┌──────────────────────┐
                │  Tool Layer          │
                └──────────┬───────────┘
                           │
                ┌──────────────────────┐
                │  Orchestration       │
                │  (Worker / GM)       │
                └──┬─────────────────┬─┘
                   │                 │
                   ▼                 ▼
           ┌──────────────┐   ┌──────────────┐
           │  Repository  │   │  Engine      │
           │  (I/O)       │   │  (순수)      │
           └──────────────┘   └──────────────┘
                   │
                   ▼
            (DB / Redis 등)
```

핵심:
- **Engine과 Repository는 형제(sibling)**. Engine이 Repository를 호출하지 않음
- **Orchestration이 둘 다 부르는 도구로 사용**
- Engine은 Repository의 존재를 모름 (FC/IS 핵심 원칙)

이 구조가 에러 처리/로깅 책임 분담의 출발점.

---

## 2. 도메인 실패 vs 인프라 실패

가장 중요한 구분.

| | 도메인 실패 | 인프라 실패 |
|---|------|------|
| **예** | "cash 부족", "재고 부족" | "DB 연결 끊김", "ERP가 DB에 없음" |
| **본질** | 게임 규칙 안의 정상 시나리오 | 시스템에 문제가 있는 비정상 |
| **표현** | 결과 객체 (Result) | 예외 raise |
| **호출자 처리** | `if result.success:` | `try/except` |
| **누구 잘못** | 아무도 (정상 흐름) | 코드 버그 또는 외부 장애 |

```python
# 도메인 실패: 결과 객체로
def execute_trade(seller, buyer, offer) -> TradeResult:
    if not seller.inventory.has_resource(...):
        return TradeResult(success=False, reason="insufficient_inventory")
    return TradeResult(success=True, ...)

# 인프라 실패: raise
class PostgresERPRepo:
    def get(self, erp_id):
        row = self.session.query(...).first()
        if row is None:
            raise NotFoundError(f"ERP not found: {erp_id}")
        ...
```

---

## 3. 레이어별 예외 책임

| 레이어 | 역할 | 예외에 대한 행동 |
|--------|------|----|
| **Repository (Adapter)** | 인프라와 도메인의 경계 | 인프라 라이브러리 예외(SQLAlchemyError, KeyError 등)를 도메인 예외(`RepositoryError`/`NotFoundError`)로 **번역해서 raise** |
| **Engine** | 순수 도메인 계산 | Repository를 모름. 도메인 실패는 결과 객체. **진짜 불가능한 상태만** raise (assertion-like) |
| **Orchestration (Worker/GM)** | 데이터 로드 + 엔진 호출 + 영속화 | Repository 예외 catch 가능. 재시도/로깅/위로 전파 결정 |
| **Tool Layer** | LLM과의 인터페이스 | 예외 → LLM이 이해할 수 있는 구조화 응답으로 변환 |
| **HTTP/API** | 외부 시스템 경계 | 안 잡힌 예외 → HTTP 상태 코드 매핑 (NotFound → 404, 그 외 → 500) |

핵심 원칙: **"의미 있게 처리할 수 있는 가장 가까운 곳에서만 잡는다."**

```python
# 시나리오 A: Repository NotFoundError → Tool에서 변환
def accept_offer_tool(offer_id):
    try:
        result = accept_offer_flow(offer_id, ...)  # NotFoundError 가능
    except NotFoundError:
        # ← 여기서 잡는 이유: LLM이 다음 행동 결정하려면 정보 필요
        return error(ErrorCode.OFFER_NOT_FOUND, "This offer no longer exists.")
    except RepositoryError:
        # 진짜 인프라 장애 → LLM이 어쩔 도리 없음 → 위로 전파
        raise
    return success(...)
```

---

## 4. Exception 클래스 vs Error 코드 — 다른 도구

| | Exception 클래스 | Error 코드 |
|---|---|---|
| **누가 보나** | Python 코드 (try/except) | LLM (응답 JSON) |
| **용도** | control flow (코드 분기) | contract (외부와의 계약) |
| **언제 쓰나** | catch할 때 | return할 때 |
| **만드는 법** | `class FooError(Exception): ...` | `enum`/`Literal` 문자열 |

**둘은 대체 관계가 아니라 보완 관계.** 한 함수 안에 같이 등장:

```python
try:
    offer = offer_repo.get(offer_id)  # ← Exception (control flow)
except NotFoundError:
    return error(ErrorCode.OFFER_NOT_FOUND, "...")  # ← Error code (LLM contract)
```

### 왜 둘 다 필요한가

**Exception만 쓰면**: LLM에게 던질 응답 형식이 표준화 안 됨. 어떤 에러가 가능한지 LLM이 못 앎.

**Error 코드만 쓰면**: 매 호출이 `if result.code == "...":` 보일러플레이트. Pythonic하지 않음 (Rust/Go의 Result 패턴).

```python
# Result 패턴 (Python에선 안 좋음)
result = erp_repo.get(seller_id)
if not result.success:
    if result.code == "not_found":
        ...
seller = result.erp

result = erp_repo.get(buyer_id)
if not result.success:
    ...
buyer = result.erp

# Exception (Pythonic, EAFP)
seller = erp_repo.get(seller_id)
buyer = erp_repo.get(buyer_id)
# 에러는 한 번만 catch
```

Python은 **EAFP** (Easier to Ask Forgiveness than Permission) 문화. try/except가 정상.

---

## 5. KeyError 같은 내장 예외를 그대로 쓰면?

가능하지만 **의미가 광범위해서 호출자가 구분 못 함**.

```python
def accept_offer_flow(offer_id):
    try:
        offer = offer_repo.get(offer_id)
        seller = erp_repo.get(offer.from_erp_id)
        config = game_configs[session_id]  # 여기도 KeyError 가능
        ...
    except KeyError as e:
        # 어디서 난 KeyError?
        # offer 미스? erp 미스? config dict 미스?
        # 같은 type이라 구분 불가
        ...
```

→ Adapter의 본질적 일은 **인프라 예외를 도메인 예외로 번역**하는 것:

```python
class InMemoryERPRepo:
    def get(self, erp_id):
        try:
            return self._store[erp_id]  # KeyError 발생 가능
        except KeyError:
            raise NotFoundError(f"ERP not found: {erp_id}") from None
```

---

## 6. Exception 정의 위치 — 응집도가 1순위

### 1순위: 응집도 (cohesion / 공유 범위)

- 하나의 모듈/Repository에만 종속된 예외 → **그 파일 안에 인라인**
- 여러 모듈/파일에 걸쳐 사용 → **별도 파일로 추출**

### 2순위: 가독성 (file size)

- 응집도가 같다면 파일이 너무 길어졌을 때 시각적으로 분리

### 안티패턴: 개수 기반 분리

"Exception 5개 넘으면 무조건 분리" 같은 heuristic은 잘못됐음. Exception 100개라도 한 모듈에서만 쓰이면 인라인이 더 응집도 높음.

### 실제 라이브러리 사례

**Django** — 모델별로 정의:
```python
class User(models.Model):
    class DoesNotExist(...): ...    # User만 쓰니까 User 안에
    class MultipleObjectsReturned(...): ...
```

**SQLAlchemy** — 공통 예외만 별도 모듈:
```python
# sqlalchemy/exc.py — 모든 쿼리에서 공유되니까
class NoResultFound(...): ...
class IntegrityError(...): ...
```

---

## 7. 로깅 — Signal vs Noise

### 레벨의 본질: "누가 봐야 하는지 + 얼마나 급한지"

| Level | 누가 보나 | 언제 보나 | 보존 |
|-------|----------|----------|------|
| **DEBUG** | 개발자 본인 | 로컬 디버깅 중 | 프로덕션엔 OFF |
| **INFO** | Ops 엔지니어 | 사후 분석, 흐름 추적 | 7~30일 |
| **WARNING** | Ops 엔지니어 | 주기적 리뷰, 트렌드 감시 | 30~90일 |
| **ERROR** | Ops + 개발자 | Slack 알림, 대시보드 | 90일+ |
| **CRITICAL** | 온콜 엔지니어 | **즉시 호출 (PagerDuty)** | 영구 |

핵심: **CRITICAL은 새벽 3시에 사람을 깨운다.** 남발하면 alert fatigue로 진짜 장애 놓침.

### 각 레벨의 용도

- **DEBUG**: 흐름 추적용 verbose 정보. 프로덕션 OFF
- **INFO**: 정상 작동의 마일스톤. 합치면 흐름이 보임
- **WARNING**: 비정상이지만 시스템은 동작. 트렌드 분석용
- **ERROR**: 실제 실패. 사람이 봐야 함. 코드 버그 / 외부 장애
- **CRITICAL**: 시스템 자체가 위태로움. 즉시 사람 호출

---

## 8. 어느 레이어가 로그를 찍는가

```
HTTP/API     ← request/response 로그 (INFO)
Tool         ← LLM 호출 입출력 (INFO/DEBUG)
Worker / GM  ← ★ 메인 로깅 레이어 (INFO/WARNING/ERROR)
Engine       ← 로그 X (순수 함수)
Repository   ← 최소한, 주로 ERROR만 (인프라 healthcheck 정도)
```

### 원칙 1: 순수 함수는 로그 안 찍는다

엔진이 로그를 찍으면 다른 컨텍스트 재사용 불가, 로그 인프라 의존성 생김 (FC/IS 깨짐). 엔진은 결과 객체에 정보 담고, 호출자가 그걸 보고 로그.

### 원칙 2: Repository는 최소한

Repository가 로그를 찍으면 위 레이어가 또 찍을 가능성 → 중복. 단, **커넥션 풀 고갈, 슬로우 쿼리** 같은 인프라 자체 정보는 여기서.

### 원칙 3: 같은 사건을 여러 레이어에서 로그하지 않는다

```
❌ Bad: NotFoundError가 Repository → Worker → Tool에서 각각 로그
   → Kibana에서 같은 사건이 3줄로 보임. 진짜 문제 찾기 어려움

✅ Good: 잡은 사람만 로그. 위로 전파하면 로그 X
```

**규칙**: **잡은 사람이 로그를 찍는다.** 전파만 하면 로그 X.

---

## 9. 12-factor 로그 수집

### 옛날 방식 (안 좋음)

앱이 직접 파일에 쓰기. 로테이션, 동시성, 손실 위험 다 앱이 떠안음.

### 12-factor 방식

> "Logs are an event stream. Treat them as such."

```
[App process] → stdout
       │
       ▼
[Container runtime / stdout 캡쳐]
       │
       ▼
[Log Shipper] (Fluent Bit, Vector, Promtail, Filebeat)
       │
       ▼
[Central Store] (Elasticsearch, Loki, Datadog, S3+Athena)
       │
       ▼
[UI] (Kibana, Grafana, Datadog dashboard)
```

**앱은 stdout/stderr에만 쓴다.** 로그 라우팅/포맷팅/저장은 인프라 책임.

### 왜 분리하나

1. 앱은 자기 일에만 집중
2. 앱이 죽어도 로그 유실 X (shipper가 별도 프로세스)
3. destination 변경/추가가 쉬움
4. 로컬 개발 = 그냥 stdout 보면 됨

---

## 10. Structured Logging + Trace ID

### Plain text (안 좋음)

```python
logger.info(f"Agent mill_1 woke at round 5 with cash 200")
```
→ 정규식 파싱 필요. 검색/집계 어려움.

### Structured (JSON)

```python
logger.info("agent_woken", extra={
    "agent_id": "mill_1",
    "round": 5,
    "cash": 200,
})
```
→ Loki/Elasticsearch에서 `{event="agent_woken", round=5}` 같은 쿼리 즉시.

Python에서는 **structlog** 라이브러리가 표준.

### Trace ID

분산 환경에서 한 요청이 여러 service를 거칠 때, 같은 요청의 로그를 묶기 위함.

```
{"event": "round_started", "round": 5, "trace_id": "abc123"}
{"event": "celery_dispatched", "agent": "mill_1", "trace_id": "abc123"}
{"event": "agent_woken", "agent": "mill_1", "trace_id": "abc123"}
{"event": "trade_executed", "offer": "x1", "trace_id": "abc123"}
```

`trace_id=abc123` 검색하면 그 요청의 전체 흐름이 한 번에. 표준은 **OpenTelemetry**.

---

## 11. 안티패턴 모음

### ❌ 무차별 catch

```python
try:
    erp = erp_repo.get(erp_id)
except Exception:  # 너무 넓음
    return None    # 정보 손실
```

### ❌ 잡고 같은 거 다시 raise

```python
try:
    erp = erp_repo.get(erp_id)
except NotFoundError:
    raise NotFoundError("ERP not found")  # 의미 없음
```

### ❌ Repository에서 도메인 로직 catch

```python
class ERPRepository:
    def transfer(self, from_id, to_id, amount):
        if from_erp.cash < amount:
            raise InsufficientCashError  # Repository는 도메인 규칙 모름
```
→ Repository는 데이터만, 도메인 규칙은 엔진에서.

### ❌ 엔진에서 Repository 호출

```python
def execute_trade(seller_id, buyer_id, offer):
    seller = erp_repo.get(seller_id)  # 엔진이 Repository 알면 안 됨
```
→ 엔진은 객체를 받고 객체를 뱉음.

### ❌ 모든 레이어에서 같은 사건 로깅

```
Repository: logger.error("not found")
Worker:     logger.error("not found")
Tool:       logger.error("not found")
```
→ 로그 3배. 대시보드 노이즈.

### ❌ 메시지 i18n 없이 모으기

```python
class ErrorMessages:
    INSUFFICIENT_CASH = "You don't have enough cash"
```
→ 메시지엔 보통 변수가 들어감 (`f"Need ${x}, have ${y}"`). 인라인이 정답. i18n 필요한 시점에 i18n 도구로.

---

## 12. 핵심 정리

| 주제 | 한 줄 |
|------|------|
| **도메인 vs 인프라** | 도메인 실패는 결과 객체, 인프라 실패는 raise |
| **Repository** | 인프라 라이브러리 예외를 도메인 예외로 번역 |
| **Engine** | Repository 모름. 진짜 불가능한 상태만 raise |
| **Orchestration** | 예외 catch 가능. 재시도/로깅/전파 결정 |
| **Tool** | 예외 → LLM 응답으로 변환 |
| **Exception vs Error code** | Exception = control flow, Error code = LLM contract. 둘 다 필요 |
| **Exception 위치** | 응집도 1순위, 가독성 2순위 |
| **로깅 레이어** | 잡은 사람이 한 번만 |
| **로그 수집** | 앱은 stdout만, 인프라가 라우팅 (12-factor) |
| **로그 형식** | Structured JSON + trace_id |

---

## 13. 레퍼런스

- "The Twelve-Factor App" — Logs as event streams (factor XI)
- "Clean Code" (Robert C. Martin) — Error Handling chapter
- "Release It!" (Michael Nygard) — Stability patterns
- Python EAFP: PEP 3156, 공식 docs "Errors and Exceptions"
- structlog 공식 문서
- OpenTelemetry 공식 사양 — Trace context propagation
- Django 소스 — 모델별 DoesNotExist 패턴
- SQLAlchemy 소스 — `sqlalchemy.exc` 모듈
