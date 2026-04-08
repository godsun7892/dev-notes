# Authoritative Server vs Worker + DB Transaction

| 항목 | 내용 |
|------|------|
| Date | 2026-04-08 |
| Context | AgentPit 거래 시스템 설계 — GM 중앙 처리 vs Worker 분산 처리 결정 |
| Decision | Worker + DB Transaction 채택 |

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
| 적합 대상 | 실시간 물리 동기화, FPS/액션 게임 | **원장 기반 자산 이동, 거래 시스템** |

---

## 3. AgentPit에서 Worker를 선택한 이유

### 3.1 우리 시스템은 원장 시스템이지 물리 엔진이 아니다

AgentPit의 거래 = "자원과 현금이 에이전트 간 이동". 이건 Stripe에서 결제 처리하는 것과 구조적으로 동일. 실시간 물리 동기화가 필요한 FPS 서버 패턴을 쓸 이유가 없다.

### 3.2 스케일 요구사항

```
목표: 10만+ 에이전트
라운드당 깨어나는 에이전트: ~200
에이전트당 거래: ~3-5건
라운드당 총 거래: ~1,000건

Authoritative Server: 1,000건이 한 서버 경유 → 서버 1대 스펙에 종속
Worker: 1,000건이 워커 N대에 분산 → 수평 확장
```

### 3.3 Lazy Activation과의 정합성

이미 에이전트 상태 처리를 Lazy(Orleans Virtual Actor 패턴)로 결정. 각 에이전트가 깨어날 때 자기 상태를 처리. 이 구조에서 거래만 중앙 GM을 경유하면 **아키텍처 불일치**가 발생.

```
상태 처리: 분산 (lazy) ← 이미 결정
거래 처리: 분산 (worker + DB transaction) ← 일관된 선택
NPC 정산: 중앙 (GM) ← 경쟁 배분이라 모아야 하므로 예외적으로 중앙
```

### 3.4 GM의 실제 역할 재정의

GM은 "모든 걸 처리하는 서버"가 아니라 **오케스트레이터**:

| GM이 하는 것 | GM이 하지 않는 것 |
|---|---|
| 누구를 깨울지 결정 (wake trigger) | 거래 검증/실행 (→ tool handler + DB) |
| NPC 일괄 정산 (경쟁 배분) | 소멸/생산 완료 처리 (→ agent lazy) |
| 라운드 진행 관리 | 개별 에이전트 상태 관리 |

---

## 4. 구현 시 주의점

### 4.1 Deadlock 방지

두 에이전트의 ERP를 동시에 lock할 때, 순서가 다르면 deadlock:

```
Worker 1: LOCK(A) → LOCK(B)  ← 대기
Worker 2: LOCK(B) → LOCK(A)  ← 대기 → DEADLOCK
```

해결: **agent_id 알파벳순으로 항상 lock** (canonical ordering)

```sql
SELECT * FROM erp 
WHERE agent_id IN ('agent_a', 'agent_b') 
ORDER BY agent_id 
FOR UPDATE;
```

### 4.2 Optimistic vs Pessimistic Locking

| | Pessimistic (SELECT FOR UPDATE) | Optimistic (version check) |
|---|---|---|
| 동작 | row를 잠그고 작업 | 읽을 때 version 기록, 쓸 때 version 비교 |
| 충돌 시 | 대기 (lock 대기) | 재시도 (version 불일치 → retry) |
| 적합 | 충돌 빈번 | 충돌 희소 |
| AgentPit | **같은 자원에 동시 거래가 빈번할 수 있음** | — |

→ **Pessimistic locking (SELECT FOR UPDATE)** 채택. 거래 대상 자원이 겹칠 확률이 높은 소규모 시장.

### 4.3 Transaction 범위 최소화

```python
# Bad: 트랜잭션 안에서 LLM 호출 (수 초 lock)
async with db.transaction():
    result = await call_llm(agent)  # 3초 동안 row lock ← 위험
    await update_erp(...)

# Good: LLM 호출은 밖에서, 트랜잭션은 DB 연산만
result = await call_llm(agent)
async with db.transaction():  # ms 단위
    await update_erp(...)
```

---

## 5. 참고 패턴/레퍼런스

| 시스템 | 패턴 | 관련성 |
|---|---|---|
| Stripe | 분산 워커 + PostgreSQL 트랜잭션 | 결제 = 양쪽 잔고 원자적 업데이트. 우리와 동일 구조 |
| Orleans (Halo) | Virtual Actor + turn-based activation | Lazy Activation의 근거 |
| TigerBeetle | 금융 전용 DB, 복식부기 강제 | transfer = 단일 원자적 연산 |
| EVE Online | Solar System별 독립 틱 | 비활성 영역 처리 안 함 → Lazy |
| PostgreSQL FOR UPDATE | row-level pessimistic locking | 거래 원자성 보장 메커니즘 |
