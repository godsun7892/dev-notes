# Authoritative Server vs Worker + DB Transaction

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 동시 요청을 직렬화하는 두 가지 근본 패턴 — 언제 어느 쪽을 쓰는가? |

---

## 1. 두 패턴 정의

### Authoritative Server (중앙 권한 서버)

모든 상태 변경이 하나의 서버를 경유. 클라이언트는 "요청"만 하고, 서버가 검증 + 실행.

```
Client A ─┐
Client B ─┤→ Authoritative Server → State
Client C ─┘     (모든 검증/실행)
```

- 대표: Unreal Engine Dedicated Server, Unity Netcode, WoW Realm Server
- 원래 목적: 실시간 물리/렌더링 동기화. 치팅 방지.

### Worker + DB Transaction (분산 워커)

각 워커가 독립적으로 요청을 처리. 원자성은 DB 트랜잭션이 보장.

```
Worker A ─┐
Worker B ─┤→ Database (ACID Transaction)
Worker C ─┘
```

- 대표: Stripe, PayPal, 은행 코어뱅킹, 모든 핀테크
- 원래 목적: 원장(ledger) 기반 자산 이동. 수평 확장.

---

## 2. 비교

| 기준 | Authoritative Server | Worker + DB Transaction |
|------|---------------------|------------------------|
| 직렬화 방식 | 서버가 순서 보장 (single thread or queue) | DB lock이 순서 보장 |
| 확장성 | 수직 확장만 (서버 1대) | 수평 확장 (워커 N대) |
| SPOF | 서버 = SPOF | DB가 SPOF (but DB는 HA 구성 가능) |
| 지연 | 모든 요청이 한 곳 경유 → 병목 | 워커별 독립 처리 → 병렬 |
| 복잡도 | 로직 한 곳 → 단순 | 동시성 제어 필요 (lock 설계) |
| 적합 대상 | 실시간 물리 동기화, FPS/액션 게임 | 원장 기반 자산 이동, 거래 시스템 |

---

## 3. 패턴 선택 기준

### Authoritative Server를 선택해야 하는 경우
- **실시간 물리 시뮬레이션**: 모든 클라이언트가 같은 물리 상태를 공유해야 함 (FPS, 격투 게임)
- **치팅 방지가 핵심**: 클라이언트를 신뢰할 수 없음, 서버만이 진실
- **렌더링 프레임 동기화**: 60fps 단위로 일관된 상태가 필요
- **상태 변화가 매우 빠르고 작음**: 위치/회전 같은 연속 변수

### Worker + DB Transaction을 선택해야 하는 경우
- **원장 기반 시스템**: 잔고/재고/포인트 같이 "양쪽이 동시에 바뀌는" 자산 이동
- **수평 확장 필요**: 트래픽이 단일 서버 한계를 넘어감
- **이산 이벤트**: 거래/주문/예약 같은 분명한 단위
- **감사 추적 필요**: 모든 변경이 트랜잭션 로그로 남아야 함

### 같은 시스템이 둘을 혼합할 수도 있다
실시간 게임도 결제/인벤토리/거래는 별도 worker + DB 트랜잭션으로 처리. 물리 동기화는 authoritative server, 자산 이동은 worker. **두 패턴은 배타적이지 않다.**

---

## 4. 구현 시 주의점 (Worker + DB Transaction)

### 4.1 Deadlock 방지

두 객체를 동시에 lock할 때, 순서가 다르면 deadlock:

```
Worker 1: LOCK(A) → LOCK(B)  ← 대기
Worker 2: LOCK(B) → LOCK(A)  ← 대기 → DEADLOCK
```

해결: **canonical ordering** (항상 같은 순서로 lock)

```sql
SELECT * FROM accounts
WHERE account_id IN ('acc_a', 'acc_b')
ORDER BY account_id
FOR UPDATE;
```

ID 알파벳순/숫자순 정렬. 모든 worker가 같은 순서로 lock하면 deadlock 불가능.

### 4.2 Optimistic vs Pessimistic Locking

| | Pessimistic (`SELECT FOR UPDATE`) | Optimistic (version check) |
|---|---|---|
| 동작 | row를 잠그고 작업 | 읽을 때 version 기록, 쓸 때 version 비교 |
| 충돌 시 | 대기 (lock 대기) | 재시도 (version 불일치 → retry) |
| 적합 | 충돌 빈번 | 충돌 희소 |
| 비용 | 락 대기 시간 | 재시도 비용 |

선택 기준:
- **충돌 빈번 (같은 row에 동시 쓰기 많음)** → Pessimistic. 재시도 대신 대기
- **충돌 희소 (대부분 다른 row)** → Optimistic. 락 오버헤드 회피

```python
# Pessimistic
async with db.transaction():
    account = await db.fetch("SELECT * FROM accounts WHERE id = $1 FOR UPDATE", id)
    account.balance -= amount
    await db.execute("UPDATE accounts SET balance = $1 WHERE id = $2", account.balance, id)

# Optimistic
account = await db.fetch("SELECT * FROM accounts WHERE id = $1", id)
new_balance = account.balance - amount
result = await db.execute(
    "UPDATE accounts SET balance = $1, version = version + 1 WHERE id = $2 AND version = $3",
    new_balance, id, account.version
)
if result.rowcount == 0:
    raise ConcurrencyError()  # 다른 트랜잭션이 먼저 업데이트
```

### 4.3 Transaction 범위 최소화

트랜잭션 안에서 느린 작업(외부 API, LLM 호출, 파일 I/O)을 하면 lock이 길어진다.

```python
# ❌ Bad: 트랜잭션 안에서 외부 호출 (수 초 lock)
async with db.transaction():
    result = await external_api.call(...)  # 3초 동안 row lock
    await update_account(...)

# ✅ Good: 외부 호출은 밖에서, 트랜잭션은 DB 연산만
result = await external_api.call(...)
async with db.transaction():  # ms 단위
    await update_account(...)
```

원칙: **트랜잭션 안에서는 DB 연산만**. 다른 모든 것(외부 API, 파일, 계산)은 밖에서.

### 4.4 멱등성 (Idempotency)

워커가 같은 작업을 여러 번 받을 수 있음 (재시도, at-least-once delivery). 멱등성 없으면 중복 처리 발생.

```python
# 멱등성 키 사용
async def process_payment(payment_id, idempotency_key):
    existing = await db.fetch(
        "SELECT * FROM payments WHERE idempotency_key = $1",
        idempotency_key
    )
    if existing:
        return existing  # 이미 처리됨, 같은 결과 반환
    # 처리 로직
```

Stripe API의 `Idempotency-Key` 헤더가 표준 패턴.

---

## 5. 참고 패턴/레퍼런스

| 시스템 | 패턴 | 비고 |
|---|---|---|
| Stripe | 분산 워커 + PostgreSQL 트랜잭션 | 결제 = 양쪽 잔고 원자적 업데이트 |
| TigerBeetle | 금융 전용 DB, 복식부기 강제 | transfer = 단일 원자적 연산 |
| PostgreSQL FOR UPDATE | row-level pessimistic locking | 거래 원자성 표준 메커니즘 |
| Orleans (Halo, Microsoft) | Virtual Actor + turn-based activation | 분산 actor 모델, lazy activation |
| Akka | Actor 모델 + supervision tree | JVM의 분산 actor |
| EVE Online | Solar System별 독립 틱 | 활성 영역만 처리, 비활성 영역 무시 |
| Unreal Dedicated Server | Authoritative Server (FPS 표준) | 권한 서버 + 클라이언트 예측 |

---

## References

- [Stripe Engineering — Idempotency](https://stripe.com/blog/idempotency)
- [TigerBeetle — Why Existing Databases (Ab)use Two-Phase Commit](https://tigerbeetle.com/blog/2022-11-16-a-database-without-dynamic-memory)
- [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Microsoft Orleans — Virtual Actor Pattern](https://learn.microsoft.com/en-us/dotnet/orleans/overview)
- [Martin Kleppmann — Designing Data-Intensive Applications](https://dataintensive.net/) — Chapter 7 (Transactions), Chapter 8 (Distributed Systems)
- [Unreal Engine — Networking and Multiplayer](https://docs.unrealengine.com/5.0/en-US/networking-and-multiplayer-in-unreal-engine/)
- [Gabriel Gambetta — Fast-Paced Multiplayer (Authoritative Server)](https://www.gabrielgambetta.com/client-server-game-architecture.html)
