# Deadlock 방지 — 자원 순서화 (Resource Ordering)

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 한 트랜잭션에서 여러 row 를 잠글 때, 어떻게 해야 서로 다른 TX 끼리 deadlock 에 빠지지 않을까? |
| 결론 | **모든 TX 가 전역적으로 동일한 순서로 락을 획득하면 순환 대기 그래프가 구조적으로 불가능** → deadlock 자체가 발생하지 않는다. 우리 accept_offer 는 `agent_id` 오름차순으로 ERP 락을 잡는다. |
| 관련 문서 | [postgresql-locking-deep-dive.md §7](postgresql-locking-deep-dive.md) — PG 레벨 deadlock 개요. 이 문서는 **애플리케이션 레벨 방지 레시피** 각도. |

---

## 1. Deadlock 이 발생하는 구조적 조건

Coffman 조건 4개 중 하나라도 없으면 deadlock 불가:

| # | 조건 | 예 |
|---|---|---|
| 1 | Mutual exclusion | `FOR UPDATE` 는 배타 — 한 번에 한 TX 만 소유 |
| 2 | Hold and wait | TX 가 락을 쥔 채 다음 락을 기다림 |
| 3 | No preemption | DB 가 강제로 뺏지 않음 (PG 는 deadlock detector 가 희생자 선정 후 abort) |
| 4 | **Circular wait** | TX_A → TX_B 락 대기 + TX_B → TX_A 락 대기 |

DB 트랜잭션 세계에선 1~3 은 이미 고정이라 바꿀 수 없다. **개발자가 손댈 수 있는 건 오직 4번** — 순환 대기를 막는 것. 이게 "자원 순서화" 의 이론적 근거.

---

## 2. 우리 시스템에서 어디가 risky 한가

한 TX 가 **서로 다른 row 2개 이상을 `FOR UPDATE`** 로 잠그는 자리. 현재 코드에선:

| 자리 | 잠그는 row | 다른 TX 와 경합? |
|---|---|---|
| `accept_offer` (offer_tools.py) | offer 1개 + seller ERP + buyer ERP | ✅ 경합 가능 |
| `erp_repo.save` (DELETE + bulk INSERT) | 단일 ERP + 그 하위 batch | ❌ (ERP 단위 격리) |
| `channel_repo.create` | 새 channel + 새 members | ❌ (모두 신규 INSERT, 기존 row 락 없음) |
| `offer_repo._mark` | 단일 offer | ❌ |
| `message_repo.create` | 신규 message | ❌ |

즉 **지금 이 순간 `accept_offer` 가 유일한 위험 지점**. 이 한 자리만 제대로 처리하면 워커 전체가 deadlock-free.

---

## 3. `accept_offer` 에서 왜 교차 락이 생기는가

두 에이전트 A, B 가 서로에게 offer 를 보낸 상태:

- `offer_1`: proposer=A, counterpart=B, side=sell → seller=A, buyer=B
- `offer_2`: proposer=B, counterpart=A, side=sell → seller=B, buyer=A

A 가 `offer_2` 를, B 가 `offer_1` 을 거의 동시에 accept:

```
TX_A (accept offer_2)              TX_B (accept offer_1)
──────────────────────            ──────────────────────
1. FOR UPDATE offer_2   ✓          1. FOR UPDATE offer_1   ✓
2. FOR UPDATE ERP(B=seller) ✓      2. FOR UPDATE ERP(A=seller) ✓
3. FOR UPDATE ERP(A=buyer)  ⏳     3. FOR UPDATE ERP(B=buyer)  ⏳
                   │                                  │
                   └──── 서로 기다림 ─────────────────┘
                           ▼
                   PG deadlock_detector → 한쪽 40P01 abort
```

핵심 문제: TX_A 는 `B → A` 순으로, TX_B 는 `A → B` 순으로 잠근다. **offer.side 에 따라 순서가 결정돼서 전역 일관성이 없다**.

Offer row FOR UPDATE 는 이 시나리오를 못 막는다 — 두 TX 가 **서로 다른 offer** 를 잠그기 때문.

---

## 4. 해법 — "무조건 `agent_id` 오름차순"

규칙 한 줄: **두 ERP 를 잠글 때 `agent_id` 가 작은 쪽 먼저**. offer.side / seller / buyer 는 무시.

```
TX_A (accept offer_2)              TX_B (accept offer_1)
──────────────────────            ──────────────────────
1. FOR UPDATE offer_2   ✓          1. FOR UPDATE offer_1   ✓
2. FOR UPDATE ERP(A) ← 작은 쪽      2. FOR UPDATE ERP(A) ⏳ 대기
3. FOR UPDATE ERP(B) ← 큰 쪽        (TX_A 끝난 뒤)
                                   2. FOR UPDATE ERP(A) ✓
                                   3. FOR UPDATE ERP(B) ✓
```

두 TX 모두 `A → B` 순 → 순환 불가능. TX_B 는 잠깐 대기 후 통과. **대기는 일어나도 deadlock 은 일어나지 않는다**.

### 실제 코드 (apps/worker/src/worker/tools/offer_tools.py `accept_offer` TX 블록)

```python
seller_id = fresh_offer.seller_agent_id
buyer_id = fresh_offer.buyer_agent_id
first_id, second_id = sorted([seller_id, buyer_id])
first_erp  = await erp_repo.get_for_update_by_agent(first_id,  session_id=session_id)
second_erp = await erp_repo.get_for_update_by_agent(second_id, session_id=session_id)
fresh_seller = first_erp if first_id == seller_id else second_erp
fresh_buyer  = first_erp if first_id == buyer_id  else second_erp
```

2단계 구조:
1. **획득 순서** 는 `sorted([...])` 로 결정적으로 고정
2. **역할 매핑** (누가 seller 인가) 은 id 로 사후 복구. 뒤 로직이 `fresh_seller`/`fresh_buyer` 이름만 보고 돌아가서 다른 코드는 수정 불필요

---

## 5. 정렬 키 선정 시 주의 — "전역적으로 안정적" 이어야 함

정렬 키가 쓸모 있으려면:

| 요구 | 이유 | 우리 선택 |
|---|---|---|
| **전역 비교 가능** | 두 TX 가 독립적으로 계산해도 같은 순서 나와야 함 | `agent_id` (UUID 문자열) ✅ |
| **불변** | TX 중간에 값이 바뀌면 순서도 바뀜 | agent_id 는 절대 불변 ✅ |
| **모든 참여자에게 동일 정의** | "내 쪽은 버전, 네 쪽은 타임스탬프" 같은 비대칭 금지 | UUID 문자열 비교 ✅ |

PK / 자연 식별자 / 생성 타임스탬프 중 **불변하고 비교 가능한 것** 을 쓴다. 랜덤 해시나 메모리 주소처럼 TX 마다 달라지는 건 쓸 수 없다.

---

## 6. 대안 비교

### A) Optimistic (version column + retry)
- 락을 잡지 않고 version 을 읽은 뒤, UPDATE 때 `WHERE version = ?` 로 체크
- 충돌 시 rowcount=0 → 재시도
- 장점: lock 획득 대기 없음, deadlock 무관
- 단점: 경합 많으면 retry 지옥, 다중 write 원자성을 프로그래머가 코드로 짜야 함
- 우리 상황엔 부적합: `execute_trade` 는 seller+buyer ERP 를 한 트랜잭션 원자로 수정해야 하고, retry 로직이 비즈 로직을 복잡하게 만듬

### B) Advisory Lock (`pg_advisory_xact_lock`)
- row 락 대신 정수 키 하나로 논리적 락
- `pg_advisory_xact_lock(hash(min_agent_id))` 같은 식
- 장점: row 구조와 무관한 임의 락
- 단점: 해시 충돌 시 엉뚱한 TX 가 서로 대기, agent_id 가 늘어나면 관리 복잡
- PG 특화라 DB 이식성 떨어짐. **복잡도 대비 이득 없음** — 우리처럼 row 가 명확한 경우엔 `FOR UPDATE` + 정렬이 간단

### C) 재시도 기반 (단순 catch + tenacity)
- deadlock 을 허용하되 `40P01` 나면 자동 재시도
- 장점: 락 순서 신경 쓸 필요 없음
- 단점: **통계적 해결**. 경합 많으면 재시도 폭주 → 응답 지연, 사용자 경험 저하
- 우리는 **이걸 추가로 쓴다** (`TransientRepositoryError` + XAUTOCLAIM). 단 "주된 방어" 는 순서화, "안전망" 이 재시도

### D) 큰 락 범위화 (테이블 락)
- `LOCK TABLE erp IN EXCLUSIVE MODE`
- 장점: 확실함
- 단점: 동시성 0. MVP 에선 유혹적이지만 10만 agent 스케일에선 즉시 병목
- **절대 쓰지 않는다**

### 결론: **순서화 (D 대안 아님) + 안전망 재시도 (C)** 의 조합이 우리 선택.

---

## 7. 우리 프로젝트에서 앞으로 주의할 지점

다음 자리에서도 같은 규칙을 적용해야 한다 (아직 미구현):

| 예정 자리 | 왜 | 규칙 |
|---|---|---|
| GM 라운드 정산 (`apps/gamemaster`) | 여러 ERP 동시 FOR UPDATE | agent_id 오름차순 |
| 다자 협상 (1v1 → N-party) | N 명의 ERP 를 동시에 락 | 전 참여자 agent_id 오름차순 |
| 생산 주문 정산 (`settle_production`) | 여러 agent 의 resource_batch | erp_id 오름차순 |
| 채널 membership 대량 수정 | 여러 channel_id × agent_id | (channel_id, agent_id) 사전식 정렬 |

**일반 원칙**: 한 TX 안에서 여러 row 를 락으로 잠글 때는 **락 대상 목록을 먼저 정렬한 뒤 순차 획득**. 절대 "자연스러운 순서" (역할/등장 순) 로 잡지 않는다.

---

## 8. 어떻게 검증하나

데드락 재현 테스트는 타이밍 민감해서 flaky 하기 쉽다. 현실적 접근:

1. **코드 리뷰 체크리스트**: "이 TX 에서 FOR UPDATE 하는 row 가 2개 이상인가? 그렇다면 `sorted()` 로 획득 순서 고정했는가?"
2. **Chaos 테스트 (선택)**: 같은 offer 쌍을 N 스레드에서 교차 accept 하는 부하 테스트. 데드락 발생률을 메트릭으로
3. **PG 로그 관찰**: 프로덕션에서 `log_lock_waits=on` + `deadlock_timeout=1s`. `40P01` 발생 시 alert. 이건 **안전망** 이지 primary defense 가 아님

---

## 9. 한 줄 요약

**여러 row 를 한 TX 안에서 FOR UPDATE 할 때는, 락 획득 순서를 `sorted()` 로 고정해서 모든 TX 가 같은 순서로 잠그도록 만든다.** 순환 대기 그래프가 구조적으로 만들어질 수 없어 deadlock 자체가 발생하지 않는다. retry 는 보조 안전망일 뿐, 주된 방어는 순서화다.

---

## 참고

- [PostgreSQL docs — Explicit Locking · Deadlocks](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-DEADLOCKS)
- "Operating System Concepts" (Silberschatz) — 자원 순서화 (resource ordering) by Coffman
- [postgresql-locking-deep-dive.md §7 Deadlock](postgresql-locking-deep-dive.md) — PG 엔진 레벨 detector 동작
- `apps/worker/src/worker/tools/offer_tools.py` `accept_offer` — 실제 적용 코드
