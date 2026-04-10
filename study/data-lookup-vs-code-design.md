# Data Lookup vs Code — 언제 data dict를 쓰고 언제 코드로 두는가

| 항목 | 내용 |
|------|------|
| Date | 2026-04-10 |
| Context | GameEvent 팩토리 함수 설계 시 data/ 패턴을 적용할지 판단하면서 정리 |
| 핵심 질문 | 런타임 lookup용 data dict vs 코드 레벨 정의, 각각 언제 쓰는가? |

---

## 1. 판단 기준: "런타임에 키로 조회하는가?"

가장 근본적인 구분선은 하나다:

```
외부 입력(사용자, 설정, 다른 모듈)이 키를 줘서 → 런타임에 lookup 하는가?
```

- **YES** → data dict 패턴. 키-값 매핑이 필요하다.
- **NO** → 코드(함수, 클래스)로 직접 표현. dict에 넣을 이유가 없다.

### AgentPit 사례

```python
# ✅ data dict가 맞는 경우 — 런타임 lookup
# production.py에서 recipe_id(외부 입력)로 레시피를 조회
recipe = ALL_RECIPES[recipe_id]  # recipe_id는 에이전트가 선택한 값

# ✅ data dict가 맞는 경우 — 런타임 lookup
# expiry 값을 resource_id로 조회
expiry = ALL_RESOURCES[resource_id].expiry

# ❌ data dict가 불필요한 경우 — 코드가 직접 호출
# 엔진이 거래 실행할 때 팩토리를 직접 호출. lookup 없음.
event = trade_executed(session_id, round_number, ...)
```

---

## 2. Data Dict 패턴의 본질

data dict는 **Indirection(간접 참조) 레이어**다.

```
호출자 → [키] → dict → [값] → 사용
```

이 간접 참조가 가치를 갖는 조건:

1. **호출자가 구체적인 값을 모른다** — recipe_id만 알고, inputs/outputs/duration은 dict에서 가져와야 한다.
2. **값이 여러 곳에서 공유된다** — ALL_RESOURCES의 expiry가 production, auto_prod, trade 등 여러 엔진에서 참조된다.
3. **값이 코드 변경 없이 바뀔 수 있다** — 새 자원 추가 시 data 파일만 수정하면 엔진 코드는 그대로.

하나도 해당 안 되면 dict에 넣는 건 불필요한 간접 참조 — 복잡성만 추가된다.

---

## 3. 코드가 스키마 역할을 하는 경우

팩토리 함수의 시그니처는 그 자체로 스키마다:

```python
def trade_executed(
    session_id: str,
    round_number: int,
    agent_id: str,
    offer_id: str,      # ← 이게 properties 스키마
    seller_id: str,
    buyer_id: str,
    resource: str,
    quantity: int,
    price: int,
) -> GameEvent:
```

이걸 data dict로 옮기면:

```python
# 문자열 튜플 — 타입 정보 없음, IDE 지원 없음, 실수해도 런타임까지 모름
EventType(name="trade.executed", properties=("offer_id", "seller_id", ...))
```

**함수 시그니처가 더 강한 스키마인 이유:**
- 파라미터 빠뜨리면 호출 시점에 즉시 에러
- 타입 힌트로 IDE 자동완성 + 정적 분석
- 함수 이름이 곧 이벤트 타입 (매핑 불필요)

---

## 4. 정리: 패턴 선택 플로우

```
데이터를 사용하는 시점에서...

  외부 입력(키)으로 조회하는가?
    ├─ YES → data dict (ALL_RESOURCES, ALL_RECIPES 등)
    └─ NO
         │
         코드가 직접 생성/호출하는가?
           ├─ YES → 함수/클래스로 표현 (팩토리 함수 등)
           └─ NO  → 상수로 표현 (매직 넘버 제거 용도)
```

---

## 5. 업계 레퍼런스

### Table-Driven Methods — Code Complete (Steve McConnell)

McConnell이 "Table-Driven Methods" 챕터(Ch.18)에서 다루는 핵심:

> "테이블 기반 방법은 논리 구문(if/else)을 테이블 조회로 대체한다.
> 복잡한 논리를 단순한 조회로 바꿀 수 있을 때 사용하라."

적용: `ALL_RECIPES[recipe_id]`는 recipe_id별 if/else 분기를 dict lookup으로 대체한 것.
반례: 이벤트 팩토리는 분기가 아니라 생성이므로 테이블화 불필요.

### Data-Driven Design — Game Programming Patterns (Robert Nystrom)

게임 개발에서 데이터 주도 설계의 기준:

> "게임 디자이너가 코드 변경 없이 수정할 수 있어야 하는 것 → 데이터로 분리.
> 프로그래머만 건드리는 구조적 정의 → 코드로 유지."

- https://gameprogrammingpatterns.com/data-locality.html
- 관련 챕터: "Type Object" 패턴 — 런타임에 타입을 데이터로 정의

AgentPit 적용: 자원/레시피는 게임 밸런싱 시 자주 변경 → data. 이벤트 스키마는 코드 구조 변경 시에만 변경 → code.

### Registry Pattern — Martin Fowler

Fowler의 Registry 패턴 (Patterns of Enterprise Application Architecture):

> "Registry는 잘 알려진 인터페이스를 통해 서비스나 객체를 찾는 글로벌 객체다.
> 호출자가 구체적 구현을 모를 때 사용한다."

- https://martinfowler.com/eaaCatalog/registry.html

`ALL_RESOURCES`, `ALL_RECIPES`는 사실상 in-memory registry다.
이벤트 팩토리는 호출자가 구체적 함수를 직접 호출하므로 registry 불필요.

### Python 표준 라이브러리 사례

```python
# data lookup 패턴 — http.HTTPStatus
# 상태 코드(외부 입력)로 의미를 조회
status = HTTPStatus(404)  # → HTTPStatus.NOT_FOUND

# 팩토리 패턴 — dataclasses.make_dataclass
# 코드가 직접 호출, lookup 없음
MyClass = make_dataclass('MyClass', [('x', int), ('y', float)])
```

---

## 6. 안티패턴: 불필요한 간접 참조

```python
# ❌ 모든 걸 data dict에 넣는 실수
EVENT_HANDLERS = {
    "trade.executed": handle_trade_executed,
    "trade.failed": handle_trade_failed,
}
# → 핸들러를 직접 호출할 수 있는데 굳이 문자열 키로 간접 참조

# ✅ 다만, 외부 입력(메시지 큐 등)에서 이벤트 타입이 문자열로 올 때는 맞다
handler = EVENT_HANDLERS[incoming_event["event_type"]]  # 런타임 dispatch 필요
```

핵심: **같은 패턴이라도 사용 맥락에 따라 적합/부적합이 갈린다.**
dict dispatch가 필요한 순간은 외부에서 문자열이 들어올 때(역직렬화, 메시지 라우팅)다.
내부 코드끼리는 함수를 직접 호출하는 게 항상 낫다.
