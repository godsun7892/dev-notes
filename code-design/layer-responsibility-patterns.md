# 레이어별 책임과 검증 위치 — 도메인 / 엔진 / Worker / Repository

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 어떤 검증이 어느 레이어 몫인가? 같은 검증을 여러 레이어에서 해도 되는가? 엔진이 "과도한 책임"을 가진다는 판단은 어떻게 하는가? |
| 관련 문서 | [functional-core-engine-pattern.md](functional-core-engine-pattern.md), [exception-chaining.md](exception-chaining.md), [context-manager-transaction-ownership.md](context-manager-transaction-ownership.md), [error-handling-and-logging-principles.md](error-handling-and-logging-principles.md), [docs/design/engine-worker-responsibility-split.md](../../docs/design/engine-worker-responsibility-split.md) (프로젝트 규범) |

---

## 1. 이론 배경

### 1.1 Functional Core / Imperative Shell (Gary Bernhardt)

2012년 Destroy All Software 스크린캐스트에서 제시. 핵심 명제:

> "An imperative shell wraps and uses the functional core... Testing the functional pieces is very easy and often allows isolated testing with no test doubles, while also leading to an imperative shell with few conditionals, making reasoning about the program's state over time much easier."

- **Core**: 조건부 **많음** + 외부 의존성 **없음** — 순수 함수만 있는 중심부
- **Shell**: 조건부 **적음** + 외부 의존성 **많음** — I/O/DB/네트워크를 다루는 껍데기

**오독 주의**: "Shell에 분기가 적다"는 말이 "Core에 분기가 없다"는 뜻이 아님. 검증/분기는 **Core에 있어야** 한다. 그래야 테스트 더블 없이 테스트할 수 있고 Shell이 단순해진다.

### 1.2 DDD Aggregate Invariants (Evans, Vernon)

> "Model true invariants in consistency boundaries." — Vaughn Vernon, *Effective Aggregate Design*

Aggregate는 **자기 불변식을 스스로 지킨다**. Application Service(= 우리 Worker)가 외부 조건을 확인하고 Aggregate에 delegate. Aggregate 안에서 "빈 content Message 생성 금지" 같은 규칙은 **Message 객체 자체가** 책임진다.

반대 안티패턴 — **Anemic Domain Model**: 도메인 객체가 getter/setter만 있고 규칙은 Service 레이어에 흩어짐. Martin Fowler가 2003년 "[AnemicDomainModel](https://martinfowler.com/bliki/AnemicDomainModel.html)"에서 안티패턴으로 명명.

### 1.3 Parse, Don't Validate (Alexis King, 2019)

> "The difference between validation and parsing lies almost entirely in how information is preserved... you can think of a value of type `NonEmpty a` as... a proof that the list is non-empty."
> — [lexi-lambda.github.io](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)

검증하고 **같은 타입을 통과**시키는 게 아니라, 검증 결과를 **더 강한 타입**으로 변환해서 "검증 완료된 값"을 타입 시스템으로 증명. Haskell에선 `NonEmpty a`, `Positive Int` 같은 wrapper 타입. Python은 native 지원 부족 — `@dataclass(frozen=True) + __post_init__`이 최선의 근사.

### 1.4 Execute / CanExecute 패턴 (Vladimir Khorikov)

> "Invariants that lie within an aggregate should be kept by that aggregate at all times... Execute/CanExecute pattern... separates checking validity from state changes."
> — [enterprisecraftsmanship.com](https://enterprisecraftsmanship.com/posts/validation-and-ddd/)

- `CanExecute(...)` → 도메인 규칙 검증. 실패 사유 반환
- `Execute(...)` → 상태 전환만. `CanExecute`가 통과됐다고 전제

AgentPit의 `validate_offer` + `execute_trade` 분리가 이 패턴과 **구조적으로 동형**.

### 1.5 Impureim Sandwich (Mark Seemann)

> "Data crosses the boundaries of systems — not functions. Functions don't easily cross boundaries."
> — [blog.ploeh.dk](https://blog.ploeh.dk/2022/05/02/at-the-boundaries-applications-arent-functional/)

구조: **Impure (데이터 로드) → Pure (계산) → Impure (쓰기)**.

Worker tool이 Repository로 snapshot 로드(impure) → 엔진 호출(pure) → Repository save(impure) 하는 흐름이 정확히 이 샌드위치.

### 1.6 TOCTOU — Time-Of-Check to Time-Of-Use (CWE-367)

> "Checking a row value in one query and acting on it in a second query creates a gap..."

Lock 없는 preview 검증은 **stale 가능**. 최종 검증은 **원자 경계 안에서 재실행**해야 한다. Java `java.util.concurrent` 가이드(Goetz, *Java Concurrency in Practice*), PostgreSQL `SELECT ... FOR UPDATE`, Stripe `PaymentIntent` 모두 같은 원리.

---

## 2. AgentPit 레이어 맵

```
┌──────────────────────────────────────────────────────────────┐
│ Worker tool (apps/worker/tools/)                             │
│  - LLM 진입점, 인자 파싱                                     │
│  - Repository 조회 (snapshot)                                │
│  - preview 검증 (엔진 validate_* 호출)                        │
│  - async with pool.connection() as conn, conn.transaction(): │
│      SELECT FOR UPDATE → 엔진 validate_* 재실행 → execute_*   │
│      → 여러 repo save(conn=conn)                             │
│  - 도메인 예외 → ErrorCode dict 변환                           │
│  - GameEvent 생성 + 구조화 로깅                              │
└──────────────────────────────────────────────────────────────┘
             │                              ▲
             │ 순수 도메인 객체 주고받음    │
             ▼                              │
┌──────────────────────────────────────────────────────────────┐
│ Engine (libs/economy_sim/engines/)                           │
│  - validate_* (도메인 규칙, preview 가능)                     │
│  - execute_* (immutable 계산)                                │
│  - 반환: 새 도메인 객체. session/round/timestamp 모름         │
└──────────────────────────────────────────────────────────────┘
             │                              ▲
             │ 도메인 객체 생성/변환        │
             ▼                              │
┌──────────────────────────────────────────────────────────────┐
│ Domain model (libs/economy_sim/models/, libs/shared/)        │
│  - frozen @dataclass + __post_init__                         │
│  - 불변식 심층 방어 (잘못된 객체 생성 불가)                  │
│  - 순수 메서드 (with_consumed 등)                            │
└──────────────────────────────────────────────────────────────┘

Repository (apps/worker/adapters/postgres/) — Worker 보조 인프라 레이어
  - Protocol: libs/economy_sim/.../ports/ 또는 libs/shared/
  - Impl: psycopg3 raw SQL, conn 주입 지원
```

---

## 3. 레이어별 책임 (상세)

### 3.1 Domain Model 레이어

```python
@dataclass(frozen=True)
class Message:
    channel_id: str
    sender_id: str
    content: str
    round_number: int
    message_id: str

    def __post_init__(self) -> None:
        if not self.content.strip():
            raise ValueError("content must be non-empty")
        if self.round_number < 0:
            raise ValueError("round_number must be non-negative")
```

**책임**
- 자기 불변식을 `__post_init__`에서 검증
- 순수 immutable 메서드 제공 (`with_consumed`, `with_added`)

**목적**
- "이 타입의 인스턴스가 존재한다 ⇒ invariant 성립" 보증 (Parse Don't Validate 근사)
- 어떤 경로로 생성되든 규칙 강제 — 새 tool, 테스트 픽스처, 리팩터 중 실수 모두

**실패 시**
- `ValueError` 또는 도메인 precondition 예외 raise
- 이건 **Case 5** (코드 버그). 정상 경로에선 절대 안 터짐. 터지면 심층 방어가 발동한 것

### 3.2 Engine 레이어

```python
# Preview — 외부 호출용. lock 없이 stale 가능
def validate_offer(seller: ERP, offer: Offer, *, current_round: int) -> None:
    if not seller.inventory.has_resource(offer.resource_id, offer.quantity, current_round):
        raise InsufficientStockError(...)

# Execute — 검증 통과 전제. 계산만
def execute_trade(seller: ERP, buyer: ERP, offer: Offer, *, current_round: int) -> TradeResult:
    new_seller_inv, consumed = seller.inventory.with_consumed(...)
    ...
    return TradeResult(new_seller=..., new_buyer=...)
```

**책임**
- 도메인 규칙 검증 (`validate_*`)
- 상태 전환 계산 (`execute_*`)
- 도메인 예외 raise (구조화 필드 포함)

**금지**
- I/O, DB, Redis, 시계, 네트워크
- mutate (인자 수정)
- 세션/라운드/현재시각 자체 생성 (인자로 주입받음)

**실패 시**
- 도메인 예외 raise — Worker가 catch해서 ErrorCode로 매핑 (Case 1)

### 3.3 Worker Tool 레이어

```python
async def accept_offer(*, offer_id, buyer_agent_id, session_id, round_number,
                        pool, erp_repo, offer_repo, trade_repo):
    # (a) 경계 — LLM 입력 파싱
    if not offer_id:
        return error(ErrorCode.INVALID_INPUT, "offer_id required")

    # (b) Preview 검증 (트랜잭션 밖, stale 가능)
    offer = await offer_repo.get(offer_id)
    buyer = await erp_repo.get_by_agent(buyer_agent_id, session_id=session_id)
    try:
        validate_offer_not_expired(offer, current_round=round_number, ttl=OFFER_TTL)
        validate_accept(buyer, offer)
    except (OfferExpiredError, InsufficientCashError) as e:
        return error(to_error_code(e), str(e))

    # (c) 최종 — 트랜잭션 안에서 재검증 + 실행
    async with pool.connection() as conn, conn.transaction():
        fresh_offer  = await offer_repo.get_for_update(offer_id, conn=conn)
        fresh_buyer  = await erp_repo.get_for_update_by_agent(buyer_agent_id,
                                                              session_id=session_id, conn=conn)
        fresh_seller = await erp_repo.get_for_update(fresh_offer.from_erp_id, conn=conn)

        # TOCTOU 방지 재검증
        try:
            validate_offer_not_expired(fresh_offer, current_round=round_number, ttl=OFFER_TTL)
            validate_offer(fresh_seller, fresh_offer, current_round=round_number)
            validate_accept(fresh_buyer, fresh_offer)
        except (OfferExpiredError, InsufficientStockError, InsufficientCashError) as e:
            # preview 통과했지만 최종에서 실패 — race
            logger.info("trade.failed", extra={"reason": type(e).__name__, ...})
            return error(to_error_code(e), str(e))

        result = execute_trade(fresh_seller, fresh_buyer, fresh_offer, current_round=round_number)

        # 엔진이 모르는 필드는 여기서
        trade = Trade(
            trade_id=new_uuid7(),
            session_id=session_id,
            round_number=round_number,
            seller_agent_id=fresh_seller.agent_id,
            buyer_agent_id=fresh_buyer.agent_id,
            resource_id=fresh_offer.resource_id,
            quantity=fresh_offer.quantity,
            total_price=fresh_offer.total_price,
        )

        await erp_repo.save(result.new_seller, conn=conn)
        await erp_repo.save(result.new_buyer,  conn=conn)
        await offer_repo.mark_accepted(offer_id,
                                       decided_by_agent_id=buyer_agent_id,
                                       decided_at_round=round_number, conn=conn)
        await trade_repo.create(trade, conn=conn)

    logger.info("trade.executed", extra={"trade_id": trade.trade_id, ...})
    return success(trade_id=trade.trade_id)
```

**책임**
- 외부 경계 (LLM 인자 파싱)
- Repository 조회 + snapshot
- preview 검증 (엔진 `validate_*` 호출)
- **트랜잭션 경계 소유**
- `SELECT FOR UPDATE`로 lock
- 엔진 `validate_*` **재실행** (TOCTOU 방지)
- 엔진 `execute_*` 호출
- 엔진이 모르는 필드(세션/라운드/타임스탬프/새 UUID) 채움
- 여러 repo에 `conn=conn` 주입해서 원자적 저장
- `GameEvent` 생성 + 구조화 로깅
- 도메인 예외 → `ErrorCode` dict 변환

### 3.4 Repository 레이어

```python
class PgERPRepository:
    async def save(self, erp, *, conn=None):
        try:
            if conn is None:
                async with self._pool.connection() as c, c.transaction():
                    await self._save(erp, c)
            else:
                await self._save(erp, conn)
        except Exception as e:
            raise RepositoryError(...) from e
```

**책임**
- raw SQL 실행
- 도메인 ↔ row 매핑 (`_row_to_domain` / `_domain_to_row`)
- 인프라 예외 → `RepositoryError` wrap (Case 3/4)
- conn 주입 지원 (ADR-014 §D3)

**금지**
- 도메인 로직 (예: "재고 부족이면 거절")
- 도메인 예외 catch (Repository는 인프라만)

---

## 4. 검증 위치 판단 매트릭스

| 검증 내용 | 모델 `__post_init__` | 엔진 `validate_*` | Worker tool |
|---|---|---|---|
| 자체 불변식 (content non-empty, quantity>0) | ✅ | ⚠️ 중복 가능 | ⚠️ LLM friendly 응답 |
| 도메인 규칙 (재고, 현금) | — | ✅ | ✅ preview + 최종 재호출 |
| 외부 상태 (agent 존재, 세션 멤버) | ❌ | ❌ | ✅ |
| LLM 입력 정규화 (trim, case) | ❌ | ❌ | ✅ |
| 멀티테넌시 / 세션 격리 | ❌ | ❌ | ✅ (tool closure + session_id) |
| 인증/인가 | ❌ | ❌ | ✅ (보통은 그보다 한 레이어 위) |

### 판단 기준

1. **"도메인 지식만으로 판단 가능한가?"**
   - Yes → 모델/엔진 (DB/외부 조회 없음)
   - No → Worker

2. **"어떤 경로로든 invalid 객체가 만들어지면 안 되는가?"**
   - Yes → 모델 `__post_init__` 추가
   - No → tool에서만 충분

3. **"Worker가 알아야 할 외부 상태가 필요한가?"** (예: "이 agent가 session 멤버인지")
   - Yes → Worker tool (Repository 조회 필요)

---

## 5. 이중 안전망 (Defense in Depth)

같은 규칙을 여러 레이어에서 검증하는 게 **냉각 낭비처럼 보이지만 필수적**이다. 각 레이어가 목적이 다르기 때문.

### Content non-empty 예시

```python
# Tool 레이어 — LLM 경계
async def send_message(*, content, ...):
    if not content.strip():
        return error(ErrorCode.INVALID_INPUT, "content required")  # LLM friendly
    message = Message(content=content, ...)

# Model 레이어 — 심층 방어
@dataclass(frozen=True)
class Message:
    content: str
    def __post_init__(self):
        if not self.content.strip():
            raise ValueError(...)  # 코드 버그로 취급 (Case 5)
```

### 양쪽이 하는 일

| 레이어 | 목적 | 실패 시 의미 |
|---|---|---|
| Tool | LLM 입력이 규격 어긴 걸 정중히 거절 | 예상된 실패. `ErrorCode` dict. WARN 로그 |
| Model | 어떤 경로로든 invalid 객체 생성 차단 | 예상 못한 실패. `ValueError`. 버그. ERROR 로그 |

### 정상 경로에선 모델 검증은 절대 터지지 않는다

tool 통과 → 이미 검증된 값으로 `Message(...)` 생성. 이때 `__post_init__`은 "한 번 더 확인"이지만 실제로는 항상 pass. **심층 방어는 발동 안 하는 게 정상**.

만약 `__post_init__`이 터진다면:
- 테스트 픽스처가 빈 content로 Message 만듦 (테스트 코드 버그)
- 새 tool이 tool 레이어 검증 잊음 (리뷰에서 잡혀야 함)
- Repository가 DB row를 Message로 조립할 때 raw 값이 잘못됨 (DB 제약이 뚫림 = 인프라 문제)

어느 쪽이든 **버그 신호**. 그래서 Case 5.

### "이중 검증은 성능 낭비 아닌가?"

아님. 두 가지 이유:

1. `__post_init__`은 O(1) 연산 대부분. 성능 문제 안 됨
2. invalid 객체가 system 깊숙이 들어가서 늦게 터지는 것보다 **생성 시점에 즉시 fail**하는 게 디버깅 훨씬 쉬움

---

## 6. Bad / Good 코드 예시

### Case 1 — "엔진에 세션 로직 섞기"

❌ Bad:
```python
# engines/trade.py
def execute_trade(seller, buyer, offer, *, session_id, repository):
    trade_row = repository.load_session_config(session_id)
    ...
```

왜 나쁜가:
- 엔진이 session_id / repository를 알게 됨 → Functional Core 깨짐
- Worker 없이 엔진 테스트 불가 (Repository mock 필요)
- CLAUDE.md "economy_sim은 사용자/산업을 모름" 위반

✅ Good:
```python
# engines/trade.py
def execute_trade(seller, buyer, offer, *, current_round) -> TradeResult:
    ...  # 순수 계산만

# apps/worker/tools/
async def accept_offer(*, session_id, ..., pool, repo, ...):
    session_config = await repo.load_session_config(session_id)
    # ... 엔진 호출
```

### Case 2 — "Worker가 inventory 직접 조작"

❌ Bad:
```python
# apps/worker/tools/accept_offer.py
async def accept_offer(...):
    seller = await erp_repo.get(...)
    # inventory batch 직접 조작
    for batch in seller.inventory.holdings[offer.resource_id]:
        if batch.quantity >= offer.quantity:
            batch.quantity -= offer.quantity
            break
```

왜 나쁜가:
- Worker가 inventory batch FEFO 규칙을 알게 됨 → anemic domain model
- frozen dataclass인데 mutate 시도 (Python이 `FrozenInstanceError` 던짐)
- 규칙 변경 시 Worker 코드가 여기저기 흩어짐

✅ Good:
```python
result = execute_trade(seller, buyer, offer, current_round=round_number)
# result.new_seller.inventory가 consumed된 새 Inventory
```

### Case 3 — "lock 없이 preview 검증만"

❌ Bad:
```python
async def accept_offer(...):
    offer = await offer_repo.get(...)
    buyer = await erp_repo.get(...)
    validate_accept(buyer, offer)
    # 여기서 다른 트랜잭션이 buyer.cash 바꿀 수 있음

    new_erp = execute_trade(...)
    await erp_repo.save(new_erp)     # 검증 시점 기준으로 저장 → race
```

왜 나쁜가:
- TOCTOU — 검증과 실행 사이 race condition
- 음수 cash 발생 가능 → 도메인 불변식 영구적으로 깨짐

✅ Good: §3.3의 `accept_offer` 예시처럼 `async with conn.transaction()` 안에서 `SELECT FOR UPDATE` + 재검증.

### Case 4 — "도메인 모델이 DB를 안다"

❌ Bad:
```python
@dataclass
class Offer:
    ...
    def accept(self, db_session):
        db_session.execute("UPDATE offer SET status=...")
        db_session.commit()
```

왜 나쁜가:
- Active Record 안티패턴
- 도메인 객체를 만들려면 DB session 필요 → 테스트 지옥
- Functional Core 원칙 정면 위반

✅ Good: 도메인 객체는 immutable data + pure method만. 영속성은 Repository가 따로.

### Case 5 — "Repository가 도메인 로직"

❌ Bad:
```python
class PgOfferRepository:
    async def mark_accepted(self, offer_id, ...):
        offer = await self._load(offer_id)
        if offer.status != "sent":
            raise InvalidStateError("Can only accept sent offers")   # 도메인 규칙
        ...
```

왜 나쁜가:
- Repository는 인프라만 — 도메인 규칙 판단은 엔진/worker

✅ Good:
```python
# Worker에서 validate_*로 상태 확인 후 호출
if fresh_offer.status != "sent":
    raise OfferAlreadyDecidedError(...)
await offer_repo.mark_accepted(offer_id, conn=conn)   # Repository는 UPDATE만
```

---

## 7. "Validation vs Authentication vs Normalization"

혼동되기 쉬운 세 가지 분리:

| 구분 | 정의 | 레이어 |
|---|---|---|
| **Validation** | 값이 **규칙에 맞는가** | 모델 `__post_init__` + 엔진 `validate_*` + Worker preview |
| **Authentication / Authorization** | 행위자가 **누구이고 권한이 있는가** | API gateway / Worker tool closure (session_id 주입) |
| **Normalization** | 값을 **정규 형태로 변환** (trim, lowercase) | Worker tool boundary |

LLM이 `" Alice "`를 보냈다면:
- Normalization: trim → `"Alice"` (Worker가 함)
- Validation: non-empty? (Worker + Model)
- Authorization: "이 agent가 세션 멤버?" (Worker가 Repository 조회)

---

## 8. 이 프로젝트 적용

### Trade (재설계 중)
- 모델: `Offer`, `Trade` frozen dataclass. `__post_init__`에 quantity>0, total_price>0 등
- 엔진: `engines/trade.py`의 `validate_*` + `execute_trade` + `create_counter_offer`
- Worker tool: `send_offer`, `accept_offer`(= execute_trade 소비), `reject_offer`, `counter_offer`

### Messaging ([ADR-013](../../docs/adr/013-messaging-shared-lib.md))
- 모델: `Channel`, `Message`. `__post_init__`에 channel_type 불변식, member 수, content non-empty
- **엔진 레이어 없음** (game-independent 인프라)
- Worker tool: `create_channel`, `send_message`, `read_channel`

### ERP
- 모델: `ERP`, `Inventory`, `ResourceBatch`. `Inventory.with_consumed`가 precondition guard
- 엔진: `engines/production.py`, `engines/round.py` 등
- Worker tool: `read_my_erp`, `place_production_order` 등

---

## 9. 요약

| 키워드 | 한 줄 |
|--------|------|
| Functional Core / Imperative Shell | 검증/분기는 Core, I/O는 Shell. Shell이 Core를 wrap |
| DDD Aggregate Invariant | 도메인 규칙은 도메인 모델이 지킨다. Service는 외부 조건만 |
| Parse Don't Validate | 검증 후 **더 강한 타입**으로 변환. Python은 `frozen @dataclass + __post_init__` 근사 |
| Execute / CanExecute | `validate_*` + `execute_*` 분리. AgentPit 구조와 동형 |
| Impureim Sandwich | 경계(Shell)에서 I/O, 중간에서 순수 계산. Worker → Engine → Worker |
| TOCTOU | lock 없는 preview는 stale 가능. **트랜잭션 안에서 재검증 필수** |
| 이중 안전망 | tool(LLM friendly) + 모델(심층 방어) 양쪽 검증. 정상 경로에선 모델 검증은 터지지 않음 |

---

## References

### 원론
- [Gary Bernhardt — Functional Core, Imperative Shell](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell) — 패턴 원 제안
- [Mark Seemann — At the boundaries, applications aren't functional](https://blog.ploeh.dk/2022/05/02/at-the-boundaries-applications-arent-functional/) — Impureim Sandwich
- [Alexis King — Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — 타입으로 보증
- [Vaughn Vernon — Effective Aggregate Design](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_1.pdf) — Aggregate invariant

### 검증 패턴
- [Vladimir Khorikov — Validation and DDD](https://enterprisecraftsmanship.com/posts/validation-and-ddd/) — Execute/CanExecute
- [Martin Fowler — Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html) — 안티패턴
- [Scott Wlaschin — Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/) — Result 타입 대안

### 동시성 / 트랜잭션
- [CWE-367 — TOCTOU](https://cwe.mitre.org/data/definitions/367.html) — 공식 정의
- [Brian Goetz — Java Concurrency in Practice](https://jcip.net/) — check-then-act 패턴
- [PostgreSQL — SELECT FOR UPDATE](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE) — row-level lock

### 프로젝트 규범
- [docs/design/engine-worker-responsibility-split.md](../../docs/design/engine-worker-responsibility-split.md) — AgentPit 규범
- [docs/adr/014-state-oriented-persistence-pivot.md](../../docs/adr/014-state-oriented-persistence-pivot.md) §D3 — conn 주입 A 패턴
- [functional-core-engine-pattern.md](functional-core-engine-pattern.md) — 인자 mutate 금지
- [context-manager-transaction-ownership.md](context-manager-transaction-ownership.md) — 트랜잭션 오너십
- [exception-chaining.md](exception-chaining.md) — `from e` / `from None`
- [error-handling-and-logging-principles.md](error-handling-and-logging-principles.md) — 에러 5분류
