# Data Lookup vs Code — 언제 data dict를 쓰고 언제 코드로 두는가

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 런타임 lookup용 data dict vs 코드 레벨 정의, 각각 언제 쓰는가? |

---

## 1. 판단 기준: "런타임에 키로 조회하는가?"

가장 근본적인 구분선은 하나다:

```
외부 입력(사용자, 설정, 다른 모듈)이 키를 줘서 → 런타임에 lookup 하는가?
```

- **YES** → data dict 패턴. 키-값 매핑이 필요하다.
- **NO** → 코드(함수, 클래스)로 직접 표현. dict에 넣을 이유가 없다.

### 예시

```python
# ✅ data dict가 맞는 경우 — 런타임 lookup
# 외부 입력(사용자가 선택한 product_id)으로 상품 정보 조회
product = ALL_PRODUCTS[product_id]

# ✅ data dict가 맞는 경우 — 런타임 lookup
# 외부 입력(country code)으로 통화 단위 조회
currency = COUNTRY_CURRENCY[country_code]

# ❌ data dict가 불필요한 경우 — 코드가 직접 호출
# 코드 안에서 정해진 함수를 직접 호출. lookup 없음.
event = order_placed(user_id, product_id, quantity)
```

---

## 2. Data Dict 패턴의 본질

data dict는 **Indirection(간접 참조) 레이어**다.

```
호출자 → [키] → dict → [값] → 사용
```

이 간접 참조가 가치를 갖는 조건:

1. **호출자가 구체적인 값을 모른다** — `product_id`만 알고, 가격/이름/카테고리는 dict에서 가져와야 한다.
2. **값이 여러 곳에서 공유된다** — 같은 상품 정보를 주문/결제/배송/리포트가 모두 참조한다.
3. **값이 코드 변경 없이 바뀔 수 있다** — 새 상품 추가 시 데이터 파일만 수정하면 코드는 그대로.

하나도 해당 안 되면 dict에 넣는 건 불필요한 간접 참조 — 복잡성만 추가된다.

---

## 3. 코드가 스키마 역할을 하는 경우

팩토리 함수의 시그니처는 그 자체로 스키마다:

```python
def order_placed(
    user_id: str,
    product_id: str,
    quantity: int,
    unit_price: int,
    currency: str,
) -> Event:
    ...
```

이걸 data dict로 옮기면:

```python
# 문자열 튜플 — 타입 정보 없음, IDE 지원 없음, 실수해도 런타임까지 모름
EventType(name="order.placed", properties=("user_id", "product_id", "quantity", ...))
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
    ├─ YES → data dict (ALL_PRODUCTS, COUNTRY_CURRENCY 등)
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

적용: `ALL_PRODUCTS[product_id]`는 product_id별 if/else 분기를 dict lookup으로 대체한 것.
반례: 이벤트 팩토리는 분기가 아니라 생성이므로 테이블화 불필요.

### Data-Driven Design — Game Programming Patterns (Robert Nystrom)

게임 개발에서 데이터 주도 설계의 기준:

> "디자이너가 코드 변경 없이 수정할 수 있어야 하는 것 → 데이터로 분리.
> 프로그래머만 건드리는 구조적 정의 → 코드로 유지."

- https://gameprogrammingpatterns.com/data-locality.html
- 관련 챕터: "Type Object" 패턴 — 런타임에 타입을 데이터로 정의

기준: 자주 튜닝되는 값(밸런싱, 가격, 카탈로그)은 data. 코드 구조 변경 시에만 변경되는 정의는 code.

### Registry Pattern — Martin Fowler

Fowler의 Registry 패턴 (Patterns of Enterprise Application Architecture):

> "Registry는 잘 알려진 인터페이스를 통해 서비스나 객체를 찾는 글로벌 객체다.
> 호출자가 구체적 구현을 모를 때 사용한다."

- https://martinfowler.com/eaaCatalog/registry.html

대부분의 in-memory `ALL_*` 상수가 사실상 registry다.

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
    "order.placed": handle_order_placed,
    "order.cancelled": handle_order_cancelled,
}
# → 핸들러를 직접 호출할 수 있는데 굳이 문자열 키로 간접 참조

# ✅ 다만, 외부 입력(메시지 큐 등)에서 이벤트 타입이 문자열로 올 때는 맞다
handler = EVENT_HANDLERS[incoming_event["event_type"]]  # 런타임 dispatch 필요
```

핵심: **같은 패턴이라도 사용 맥락에 따라 적합/부적합이 갈린다.**
dict dispatch가 필요한 순간은 외부에서 문자열이 들어올 때(역직렬화, 메시지 라우팅)다.
내부 코드끼리는 함수를 직접 호출하는 게 항상 낫다.

---

## 7. 핵심 정리

| 상황 | 선택 |
|------|------|
| 외부 입력 키로 조회 | data dict |
| 코드가 직접 호출 | 함수/클래스 |
| 자주 튜닝되는 값 | data (외부 파일/DB) |
| 코드 구조의 일부 | code |
| 메시지 큐에서 string으로 dispatch | data dict (registry) |
| 내부 모듈끼리 호출 | 직접 import |

---

## 8. References

- Steve McConnell — *Code Complete*, Chapter 18 (Table-Driven Methods)
- Robert Nystrom — [*Game Programming Patterns*](https://gameprogrammingpatterns.com/) — "Type Object", "Data Locality"
- Martin Fowler — [Registry Pattern](https://martinfowler.com/eaaCatalog/registry.html)
- Martin Fowler — [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
- Python `http.HTTPStatus` — stdlib registry 사례
