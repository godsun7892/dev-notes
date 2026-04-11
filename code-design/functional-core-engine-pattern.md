# Functional Core 엔진 패턴 — 인자 mutate 금지

economy_sim 엔진이 지켜야 할 호출 규약. trade/production/auto_prod/npc 등
모든 도메인 엔진에 동일하게 적용된다.

## 한 줄 요약

엔진은 받은 객체를 절대 mutate하지 않는다. 성공이면 새 객체를 담은 결과를
리턴하고, 실패면 실패 사유만 리턴한다. 호출자의 원본은 어떤 경우에도 무사하다.

## 규칙

1. **인자는 read-only**: 함수 본문에서 `seller.cash = ...`, `inventory.add_resource(...)`
   처럼 인자 객체의 필드/내부 상태를 바꾸는 코드를 쓰지 않는다.
2. **새 객체로 표현**: 변경된 상태는 `dataclasses.replace()` 또는 새 dataclass
   인스턴스로 만든다. Inventory처럼 내부에 list/dict를 가진 경우, 그 list/dict도
   새 것을 만들어야 한다 (`with_consumed`, `with_added` 같은 immutable-style 메서드).
3. **결과 객체로 신호**: 성공/실패는 `XxxResult` 객체로 표현. 성공 시 `new_seller`/
   `new_buyer` 같은 새 객체 필드에 결과를 담는다. 실패 시엔 reason/error_code만
   채우고 새 객체 필드는 비운다.
4. **원본 "돌려주기" 같은 개념 없음**: 엔진이 인자를 안 건드리니까, 호출자는
   원본 참조를 계속 들고 있다. 잃은 적이 없으므로 돌려받을 필요가 없다.

## 시그니처 예시 (trade)

```python
@dataclass
class TradeResult:
    success: bool
    reason: str = ""
    new_seller: ERP | None = None
    new_buyer: ERP | None = None
    events: list[GameEvent] = field(default_factory=list)


def execute_trade(seller: ERP, buyer: ERP, offer: Offer) -> TradeResult:
    # 검증
    if not seller.inventory.has_resource(offer.resource, offer.quantity, offer.round_number):
        return TradeResult(success=False, reason="자원 부족")
    if buyer.cash < offer.price:
        return TradeResult(success=False, reason="현금 부족")

    # 새 inventory 만들기 (원본 안 건드림)
    new_seller_inv, consumed = seller.inventory.with_consumed(
        offer.resource, offer.quantity, offer.round_number
    )
    new_buyer_inv = buyer.inventory.with_added(consumed)

    # 새 ERP 만들기
    new_seller = replace(seller, cash=seller.cash + offer.price, inventory=new_seller_inv)
    new_buyer  = replace(buyer,  cash=buyer.cash  - offer.price, inventory=new_buyer_inv)

    return TradeResult(
        success=True,
        new_seller=new_seller,
        new_buyer=new_buyer,
        events=[...],
    )
```

## 호출자(워커) 사용법

```python
seller = erp_repo.get(offer.from_erp_id)
buyer  = erp_repo.get(offer.to_erp_id)

result = execute_trade(seller, buyer, offer)

if not result.success:
    # 원본 seller/buyer는 호출 전과 동일. "복원" 코드 필요 없음.
    return tool_error(result.reason)

# 성공 — 새 객체를 저장
erp_repo.save(result.new_seller)
erp_repo.save(result.new_buyer)
event_repo.append(result.events)
```

성공 분기에서 옛 `seller`/`buyer`는 더 이상 안 쓰이고 GC된다. 실패 분기에서는
원본을 자유롭게 사용한다 (예: 에러 응답에 현재 잔액 표시).

## 트랜잭션은 어디에 있나

엔진 안에는 없다. **호출자(워커)가 트랜잭션을 가진다.**

```python
with db.transaction():                       # ← 트랜잭션은 워커
    seller = erp_repo.get(...)               # ← DB는 워커
    buyer  = erp_repo.get(...)

    result = execute_trade(seller, buyer, offer)   # ← 엔진. DB/트랜잭션 모름

    if not result.success:
        return tool_error(result.reason)     # ← 자동 롤백

    erp_repo.save(result.new_seller)         # ← DB는 워커
    erp_repo.save(result.new_buyer)
# commit
```

엔진은 `import psycopg`, `import redis`, `db.transaction()` 어느 것도
부르지 않는다. CLAUDE.md "economy_sim은 DB/Redis/timezone 의존 X" 하드 제약은
이 분리에 의해 자동으로 만족된다.

## 비유

- **economy_sim** = 회계사. "이 거래 적법한가? 그러면 잔액이 얼마가 되는가?"
  종이에 계산만 한다. 금고에 손 안 댄다.
- **apps/worker** = 금고지기. 회계사 보고서를 받아 진짜 금고를 열고 돈을 옮긴다.
  중간에 비상벨이 울리면(예외) 금고 닫고 옛 장부 그대로 둔다 (트랜잭션 롤백).

회계사는 금고 비밀번호를 모른다. 알면 안 된다.

## 이 패턴이 막아주는 4가지 함정

`execute_trade`가 인자를 mutate하던 예전 패턴은 다음 함정을 가졌다. immutable
패턴으로 가면 4개 모두 자동으로 사라진다.

### 함정 A — 같은 객체 두 번 호출 시 결과 달라짐

```python
seller = repo.get(...)
buyer  = repo.get(...)

preview = execute_trade(seller, buyer, offer)   # mutate 발생
# ... 사용자 확인 ...
result = execute_trade(seller, buyer, offer)    # 두 번째: seller 자원 이미 빠짐
```

mutate 버전: idempotent하지 않다. 호출자가 "두 번 부르면 안 됨"을 외워야 한다.
immutable 버전: 몇 번을 불러도 동일한 결과. 원본을 안 건드리므로.

### 함정 B — 부분 실패 후 좀비 객체

```python
with db.transaction():
    result = execute_trade(seller, buyer, offer)   # mutate 완료
    erp_repo.save(seller)
    erp_repo.save(buyer)                            # ← 여기서 PG 제약 위반 raise
# 트랜잭션 롤백 → PG는 깨끗
# 그러나 메모리의 seller/buyer는 mutate된 좀비 상태
```

함수 밖에서 그 좀비 객체를 또 쓰면 (예: 에러 응답에 잔액 표시) 잘못된
값을 보게 된다. PG와 메모리의 상태가 어긋난다. immutable 버전은 원본이
손상되지 않으므로 좀비가 발생하지 않는다.

### 함정 C — 캐시/재시도와 만나면 폭발

ERP를 Redis에 캐싱하거나 in-process 재시도를 도입하는 순간, 한 번 mutate된
객체가 캐시에 살아남아 재시도 시 또 mutate된다. 자원/cash가 한 번 더 빠진다.
immutable 패턴이면 캐시된 객체가 절대 변하지 않으므로 안전하다.

### 함정 D — 테스트와 프로덕션의 의미가 어긋남

mutate 버전 단위 테스트는 인자의 상태를 검사한다. 실제 워커는 매번 PG에서
새로 로드하므로 첫 호출의 mutate가 두 번째에 영향을 안 준다. 즉 단위 테스트
환경과 실제 실행 환경의 의미가 달라진다. "테스트는 그린인데 프로덕션은 빨강"
패턴의 원인. immutable 버전은 테스트가 result를 검사하게 강제되므로 의미가 일치한다.

## 적용 대상 (현재 economy_sim)

다음 엔진들이 이 패턴으로 통일되어야 한다. 지금은 모두 in-place mutation을 한다.

- `engines/trade.py` — execute_trade
- `engines/production.py` — start_production, complete_production
- `engines/auto_prod.py` — process_auto_production
- `engines/npc.py` — settle_npc_sales, sell_intermediate_to_npc

리팩토링 비용이 가장 싼 시점은 **워커가 아직 0줄인 지금**이다. 호출자
변경 비용이 0이다.

## 관련 문서

- CLAUDE.md "Functional Core / Imperative Shell" 하드 제약
- CLAUDE.md "Engine은 객체 받고 객체 반환" 안티패턴
- dev-notes/code-design/error-handling-and-logging-principles.md — 에러 5분류와 결과 객체 패턴
