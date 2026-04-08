# 거래 실행 패턴 — 레퍼런스 및 조사

| 항목 | 내용 |
|------|------|
| Date | 2026-04-08 |
| Context | AgentPit 에이전트 간 거래를 안정적이고 효율적으로 처리하기 위한 조사 |
| 핵심 질문 | 양쪽 잔고를 동시에 변경하는 거래를 어떻게 원자적으로 처리하는가? |

---

## 1. 근본 원칙: 복식부기 (Double-Entry Bookkeeping)

1494년 루카 파치올리가 체계화. 500년간 모든 금융 시스템의 근간.

```
하나의 거래 = 하나의 분개(journal entry)
  차변(debit)  + 대변(credit)이 반드시 동시에 기록
  한쪽만 기록되는 건 물리적으로 불가능
```

AgentPit 적용:
```
거래: A가 B에게 밀 3개를 $30에 판매
  차변: A.inventory.wheat -= 3,  A.cash += 30
  대변: B.inventory.wheat += 3,  B.cash -= 30
  → 하나의 DB 트랜잭션
```

**양쪽이 각자 잔고를 수정하는 금융 시스템은 존재하지 않는다.** 전부 하나의 원자적 연산.

---

## 2. 실제 시스템 분석

### 2.1 Stripe — 분산 워커 + PostgreSQL

Stripe의 초기 아키텍처 (Patrick Collison 인터뷰, Increment Magazine 기고 기반):

```
결제 요청 → Load Balancer → Worker (N대)
  → Worker가 직접 PostgreSQL 트랜잭션 실행
  → 중앙 "PaymentMaster" 같은 서비스 없음
```

핵심 패턴:
```sql
BEGIN;
-- 1. 관련 계좌 row lock
SELECT * FROM balance_transactions 
WHERE account_id IN ($sender, $receiver) 
ORDER BY account_id FOR UPDATE;

-- 2. 잔고 검증
-- (application layer)

-- 3. 양쪽 잔고 업데이트 + 거래 기록
INSERT INTO balance_transactions (...);
UPDATE accounts SET balance = balance - $amount WHERE id = $sender;
UPDATE accounts SET balance = balance + $amount WHERE id = $receiver;
COMMIT;
```

- Worker가 몇 대든 DB 트랜잭션이 원자성 보장
- `FOR UPDATE`로 동시 접근 직렬화
- 초당 수천 건 처리

**AgentPit 관련성**: 우리 거래와 구조 동일. 자원+현금 이동 = 잔고 이동.

### 2.2 TigerBeetle — 금융 전용 데이터베이스

Joran Dirk Greef가 LMAX Exchange 경험 기반으로 설계. 복식부기를 DB 레벨에서 강제.

```
// 하나의 transfer = 양쪽 계좌 원자적 업데이트
create_transfer({
    id: unique_id,
    debit_account_id: seller,    // 자원 차감
    credit_account_id: buyer,    // 자원 추가
    amount: 30,
    ledger: 1,                   // 원장 구분
    code: 1,                     // 거래 유형
});
// 이 한 호출이 양쪽 잔고를 atomic하게 변경
// 한쪽만 성공하는 건 물리적으로 불가능
```

성능:
- 단일 노드에서 초당 100만+ transfer
- 6-node 클러스터로 HA 구성

**AgentPit 관련성**: 거래량이 많아질 경우 PostgreSQL에서 TigerBeetle로 원장 부분만 교체 가능. 동일 API 패턴.

### 2.3 주식 거래소 — 매칭 엔진

NYSE, NASDAQ, 암호화폐 거래소 (Binance, Coinbase):

```
주문 흐름:
  매수 주문 ─┐
  매도 주문 ─┤→ Matching Engine (중앙) → 양쪽 계좌 원자적 업데이트
  매수 주문 ─┘      (가격-시간 우선순위)
```

- 매칭 엔진은 중앙 (주문 매칭이 본질적으로 순차적)
- 하지만 계좌 업데이트는 DB 트랜잭션

**AgentPit과의 차이**: 우리는 주문 매칭이 없다. 양자 협상 → accept로 거래 성사. 중앙 매칭 엔진 불필요. NPC 정산만 "매칭" 성격 → GM이 처리.

### 2.4 MMO 게임 — EVE Online, WoW

```
거래 윈도우:
  Player A: 아이템 올림 → 확인
  Player B: 아이템 올림 → 확인
  양쪽 확인 완료 → 서버가 원자적 교환 실행
```

- 서버가 양쪽 확인 후 한 번에 교환
- 클라이언트는 "확인" 요청만, 실행은 서버
- DB 트랜잭션 사용

**AgentPit 관련성**: send(offer) = 아이템 올림, respond(accept) = 확인, Phase 3 = 서버 원자적 교환. 동일 구조.

### 2.5 오픈소스 참고 구현

**Mojaloop** (Bill & Melinda Gates Foundation 후원, 금융 포용 오픈소스 결제 시스템):
- GitHub: mojaloop/central-ledger
- 분산 워커 + 중앙 원장
- Two-phase transfer: prepare → fulfill/abort
- 국가 단위 결제 시스템에서 실제 운영 중

**Peatio** (오픈소스 암호화폐 거래소):
- GitHub: openware/peatio
- Ruby on Rails + PostgreSQL
- 거래 실행: DB 트랜잭션으로 양쪽 잔고 업데이트
- `Account.lock!` (pessimistic locking) 사용

---

## 3. 공통 패턴 추출

모든 시스템이 공유하는 거래 실행 패턴:

```
1. LOCK   — 관련 계좌/row를 잠금 (동시 접근 방지)
2. CHECK  — 잔고/재고 검증
3. EXECUTE — 양쪽 잔고 동시 변경
4. LOG    — 거래 기록 (감사 추적)
5. UNLOCK — 잠금 해제 (COMMIT/ROLLBACK)

이 5단계가 하나의 원자적 단위 (All or Nothing)
```

---

## 4. AgentPit 거래 실행 설계

### 4.1 3-Phase 검증 구조

```
Phase 1: send(offer) — Informational
  → 내 inventory 확인 → 없으면 차단 (offer 전송 안 됨)
  → 있으면 전송 (자원 lock 안 함)
  → 목적: 불필요한 메시지 방지

Phase 2: respond(accept) — Informational
  → 내 cash 확인 → 없으면 차단 (accept 불가)
  → 있으면 Phase 3으로
  → 목적: 불필요한 트랜잭션 방지

Phase 3: DB Transaction — Atomic
  → SELECT FOR UPDATE (양쪽 row lock)
  → 양쪽 잔고 재검증
  → 양쪽 동시 업데이트
  → 이벤트 기록
  → COMMIT or ROLLBACK
```

Phase 1, 2는 UX 최적화 (헛된 시도 줄이기). Phase 3만이 원자성 보장.

### 4.2 구현 코드

```python
async def execute_trade(
    offer: Offer,
    seller_id: str,
    buyer_id: str,
    round_number: int,
    db: Database,
) -> TradeResult:
    """
    Stripe 패턴: Worker가 직접 DB 트랜잭션 실행.
    GM 경유 없음. 원자성은 PostgreSQL이 보장.
    """
    # Deadlock 방지: agent_id 순서 고정
    first, second = sorted([seller_id, buyer_id])

    async with db.transaction():
        # 1. LOCK — 양쪽 ERP row lock
        rows = await db.fetch(
            "SELECT * FROM erp WHERE agent_id = ANY($1) ORDER BY agent_id FOR UPDATE",
            [first, second],
        )
        seller = find(rows, seller_id)
        buyer = find(rows, buyer_id)

        # 2. CHECK — 양쪽 잔고 검증
        if not seller.has_resource(offer.resource, offer.quantity):
            return TradeResult.failed("seller: 자원 부족")
        if buyer.cash < offer.total_price:
            return TradeResult.failed("buyer: 현금 부족")

        # 3. EXECUTE — 양쪽 동시 업데이트 (복식부기)
        await db.execute(
            "UPDATE erp_inventory SET quantity = quantity - $1 WHERE agent_id = $2 AND resource = $3",
            offer.quantity, seller_id, offer.resource,
        )
        await db.execute(
            "INSERT INTO erp_inventory (agent_id, resource, quantity, produced_at) VALUES ($1, $2, $3, $4)",
            buyer_id, offer.resource, offer.quantity, round_number,
        )
        await db.execute("UPDATE erp SET cash = cash + $1 WHERE agent_id = $2", offer.total_price, seller_id)
        await db.execute("UPDATE erp SET cash = cash - $1 WHERE agent_id = $2", offer.total_price, buyer_id)

        # 4. LOG — 이벤트 기록
        await db.execute(
            "INSERT INTO event_log (type, round, data) VALUES ($1, $2, $3)",
            "TRADE_EXECUTED",
            round_number,
            json.dumps({
                "seller": seller_id, "buyer": buyer_id,
                "resource": offer.resource, "quantity": offer.quantity,
                "price": offer.total_price,
            }),
        )

    # 5. UNLOCK — COMMIT 완료 (async with 빠져나오면 자동)
    return TradeResult.success()
```

### 4.3 실패 시나리오와 처리

| 시나리오 | 발생 시점 | 처리 |
|---|---|---|
| A가 이미 다른 곳에 팔았음 | Phase 3 CHECK | ROLLBACK → "자원 부족" 반환 |
| B가 다른 거래로 현금 소진 | Phase 3 CHECK | ROLLBACK → "현금 부족" 반환 |
| DB 장애 | Phase 3 | 트랜잭션 자동 ROLLBACK → 양쪽 변화 없음 |
| Worker 크래시 | Phase 3 중간 | PostgreSQL이 미완료 트랜잭션 자동 ROLLBACK |

**어떤 실패든 "아무 일도 안 일어난" 상태로 복구.** 이게 ACID의 A(Atomicity).

---

## 5. 핵심 교훈

1. **양쪽 잔고를 각자 수정하는 시스템은 없다.** 500년 복식부기 → DB 트랜잭션 → TigerBeetle까지 전부 하나의 원자적 연산.

2. **"중앙 서버"와 "원자적 실행"은 다른 개념이다.** 원자성은 DB 트랜잭션이 제공. 중앙 서버는 직렬화를 제공. 우리는 원자성이 필요하지 직렬화가 필요한 게 아님.

3. **거래 시스템은 핀테크 패턴이지 게임 서버 패턴이 아니다.** 자원/현금 이동 = 잔고 이동. Stripe와 동일 구조.

---

## 6. 참고 자료

| 자료 | 내용 | URL/출처 |
|---|---|---|
| Stripe Engineering | 초기 아키텍처, PostgreSQL 기반 결제 처리 | Increment Magazine, Stripe 엔지니어링 블로그 |
| TigerBeetle | 금융 전용 DB, 복식부기 강제, 100만+ TPS | tigerbeetle.com, GitHub: tigerbeetle/tigerbeetle |
| Mojaloop | 오픈소스 결제 시스템, 분산 워커 + 중앙 원장 | GitHub: mojaloop/central-ledger |
| Peatio | 오픈소스 거래소, PostgreSQL + pessimistic locking | GitHub: openware/peatio |
| PostgreSQL FOR UPDATE | row-level pessimistic locking 공식 문서 | postgresql.org/docs/current/sql-select.html |
| Martin Kleppmann | Designing Data-Intensive Applications, Ch.7 Transactions | O'Reilly |
| Pat Helland | "Life beyond Distributed Transactions" (2007) | ACM Queue |
