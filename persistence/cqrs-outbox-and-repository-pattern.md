# Repository / CQRS / Outbox 패턴 — 학습 노트

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 도메인 로직과 인프라(DB)를 어떻게 분리하고, 분산된 저장소 간 일관성을 어떻게 보장하는가? |

---

## 1. Model vs Repository

**Model**은 데이터의 모양 (what), **Repository**는 그 데이터를 꺼내고 넣는 법 (how / where).

```python
# Model — 데이터 구조만 정의, DB를 모름
@dataclass
class Account:
    account_id: str
    cash: int
    items: list[Item]

# Repository — 어떻게 저장/조회하는지
class AccountRepository(Protocol):
    def get(self, account_id: str) -> Account: ...
    def save(self, account: Account) -> None: ...
```

핵심: **Repository는 DB 테이블과 1:1이 아니다.** 도메인 객체 하나가 여러 테이블에 분산 저장될 수 있고, Repository가 그 사이를 번역(조립/분해)한다.

```
Account (도메인 객체, 중첩 구조)
   │
   │ Repository.get()  ← JOIN으로 조립
   ▼
[accounts 테이블] + [items 테이블] + ...
   │
   │ Repository.save() ← 분해해서 INSERT/UPDATE
   ▲
Account (mutated)
```

---

## 2. 아키텍처 패턴 비교

| 패턴 | 설명 | 적합 |
|------|------|------|
| **Transaction Script** | 로직, DB, 비즈니스 규칙이 한 함수에 섞임 | 간단한 CRUD, 스크립트 |
| **Active Record** (Django, Rails) | 모델 객체가 자기 자신을 저장 (`obj.save()`) | 웹 CRUD, 어드민 |
| **Layered (Service Layer)** | Controller → Service → Repository → DB | 대규모 팀, 엔터프라이즈 |
| **Functional Core / Imperative Shell** | 순수 함수 코어 + I/O 담당 셸 | 로직 복잡 + 테스트 중요 |
| **Hexagonal (Ports & Adapters)** | 도메인이 Port 선언, 모든 외부는 Adapter로 | 멀티 인터페이스 (UI/API/CLI) |
| **CQRS** | Command(쓰기)와 Query(읽기) 모델 분리 | 읽기/쓰기 패턴이 다를 때 |
| **Event Sourcing** | 이벤트가 진실의 원천, 상태는 파생 | 감사 추적, 리플레이 필요 |

### Hexagonal vs Functional Core 차이

```
Hexagonal:
  Driving Adapter → Port → Application → Port → Driven Adapter
  (모든 외부 상호작용이 Port를 통과, 코어가 Port를 선언)

Functional Core / Imperative Shell:
  Shell (I/O 담당) → Core (순수 함수) → Shell
  (코어는 외부 존재 자체를 모름, 그냥 데이터 in/out)
```

Hexagonal은 코어가 "나는 이런 게 필요해"라고 선언하고, FC/IS는 코어가 그냥 "데이터 줘 처리해줄게"라고 함. FC/IS가 더 단순하고, 코어 테스트에 mock도 필요 없음 (그냥 함수 호출).

---

## 3. 실제 ERP 시스템의 데이터 모델 패턴

세계 1위(SAP S/4HANA), 오픈소스 대표(ERPNext, Odoo) 모두 같은 3층 구조:

| 구분 | 역할 | SAP | ERPNext | Odoo |
|------|------|-----|---------|------|
| **Master Data** | 정의 (불변) | MARA | Item | product_template |
| **Transaction** | 무슨 일이 있었는지 (append-only) | MATDOC | Stock Ledger Entry | stock_move |
| **Current State** | 지금 얼마나 있는지 (파생) | MARD (on-the-fly) | Bin | stock_quant |

### SAP S/4HANA의 전환

기존 ECC: `MKPF (헤더) + MSEG (항목) + MARD (재고)` 분리
신규 NSDM: `MATDOC` 단일 테이블, MARD는 껍데기, 재고는 MATDOC에서 계산

> "이벤트가 진실, 상태는 파생" — 현대 ERP의 표준 방향

단, SAP도 순수 on-the-fly가 아니라 `MATDOC_EXTRACT`라는 사전 압축 테이블을 둠. **하이브리드.**

---

## 4. CQRS — 이벤트와 상태의 공존

```
[Command] → 이벤트 발행 → 상태 갱신
                  │
                  ▼
            Event Store (source of truth)
                  │
                  ▼
       [Materialized View / Projection]
                  │
                  ▼
              [Query] ← 빠른 조회
```

- **Event Store**: 무슨 일이 있었는지 (append-only, 순서 보장)
- **Materialized View**: 지금 상태가 어떤지 (이벤트에서 파생, 빠른 조회용)

### 왜 둘 다 필요한가

| 시나리오 | 이벤트만 | 상태만 | 둘 다 |
|---------|---------|--------|-------|
| 현재 잔액 조회 | 매번 리플레이 (느림) | 즉시 (빠름) | 즉시 ✅ |
| 과거 시점 리플레이 | 가능 | 불가능 | 가능 ✅ |
| 감사 추적 | 가능 | 불가능 | 가능 ✅ |
| 버그 시 복구 | 이벤트로 재구축 | 손실 | 재구축 가능 ✅ |

**원칙**: 이벤트와 상태가 충돌하면 이벤트가 이긴다. 상태는 언제든 이벤트로 재구축 가능해야 한다.

---

## 5. 분산 트랜잭션 문제

CQRS를 구현할 때 가장 큰 함정: **이벤트 저장소와 상태 저장소가 다른 DB면, 어떻게 원자성을 보장하나?**

```python
def transfer_funds(from_id, to_id, amount):
    UPDATE postgres.accounts ...   # ① 상태
    INSERT INTO mongo.events ...   # ② 이벤트
    # 둘 다 성공 OR 둘 다 실패 보장 불가능
```

PostgreSQL과 MongoDB는 같은 트랜잭션에 묶을 수 없음.

### 발생하는 문제

- ① 성공, ② 실패 → 상태는 바뀌었는데 이벤트 없음 → 리플레이/감사 깨짐
- ② 성공, ① 실패 → 이벤트는 있는데 상태 안 바뀜 → 클라이언트가 잘못된 상태 봄

### 해결책 4가지

1. **Outbox Pattern** ← 업계 표준
2. **Single Database** (PostgreSQL JSONB로 이벤트도 저장)
3. **Event-First** (이벤트 먼저, 상태는 idempotent 갱신)
4. **2PC / XA** (이론적 가능, 실무에선 거의 안 씀)

---

## 6. Outbox Pattern 상세

### 핵심 아이디어

> "이벤트를 바로 외부 시스템에 보내지 말고, 같은 DB의 outbox 테이블에 일단 적어놓자."

```python
def transfer_funds(from_id, to_id, amount):
    BEGIN postgres transaction
        UPDATE accounts SET cash = cash - $amount WHERE id = $from_id
        UPDATE accounts SET cash = cash + $amount WHERE id = $to_id
        INSERT INTO outbox (event_payload, published) VALUES ($event, false)
    COMMIT  ← 같은 PostgreSQL이라 원자성 보장
```

이 시점에 상태도 바뀌고 이벤트도 기록됨. **둘 다 같은 트랜잭션** → 하나라도 실패하면 둘 다 롤백.

### Relay 단계

별도 프로세스가 outbox를 폴링해서 외부 시스템(MongoDB, Kafka 등)으로 옮김:

```python
def relay_worker():
    while True:
        rows = postgres.query("SELECT * FROM outbox WHERE published=false LIMIT 100")
        for row in rows:
            mongo.events.insert_one(row.payload)
            postgres.execute("UPDATE outbox SET published=true WHERE id=?", row.id)
        sleep(1)
```

Relay가 죽어도 outbox에 데이터가 살아있음 → 재시작 시 이어서 처리. **데이터 손실 없음.**

### 구조

```
[Application]
   │
   ▼
┌─────────────────┐
│  PostgreSQL     │
│  ┌──────────┐   │
│  │ accounts │ ← 상태
│  ├──────────┤   │ (같은 트랜잭션)
│  │ outbox   │ ← 이벤트 임시
│  └──────────┘   │
└─────────────────┘
        │
        │ Relay (폴링 또는 CDC)
        ▼
┌─────────────────┐
│  MongoDB / Kafka│
│  (events)       │
└─────────────────┘
```

---

## 7. Outbox Transport 선택지

Outbox 자체는 PostgreSQL에 있어야 하지만 (트랜잭션 원자성), **relay 운반책은 자유롭게 선택 가능.**

| Transport | 처리량 | 순서 보장 | 내구성 | 운영 복잡도 |
|-----------|--------|----------|--------|------------|
| **Kafka** | 초당 수십만+ | 파티션 단위 | 강력 (디스크 복제) | 높음 |
| **Redis + Celery** | 초당 수천~만 | 기본 X (옵션 있음) | 중간 (RDB/AOF) | 낮음 |
| **RabbitMQ** | 초당 수만 | 큐 단위 | 강력 | 중간 |
| **PostgreSQL LISTEN/NOTIFY** | 낮음 | 보장 | 강력 | 매우 낮음 |
| **Debezium (CDC)** | 매우 높음 | WAL 순서 | 강력 | 높음 |

### Redis가 outbox 자체가 될 수는 없는 이유

```python
BEGIN postgres
  UPDATE accounts ...
COMMIT postgres

redis.xadd("events", ...)  ← 별도 작업, 원자성 깨짐
```

Redis는 PostgreSQL 트랜잭션에 참여 못 함. **Outbox는 무조건 상태와 같은 DB여야 함.**

### Kafka는 outbox 패턴의 필수 요소가 아님

LinkedIn/Uber가 Kafka를 쓰는 건 초당 수백만 events 처리 때문이지, outbox 패턴이 Kafka를 요구해서가 아님. Redis + Celery 같은 가벼운 transport로도 outbox 구현 가능.

---

## 8. 핵심 정리

| 개념 | 한 줄 요약 |
|------|----------|
| **Repository Pattern** | 도메인 객체와 DB 사이의 번역기 |
| **Functional Core/Shell** | 순수 함수 코어 + I/O 담당 셸 |
| **CQRS** | 쓰기(이벤트) 모델과 읽기(상태) 모델 분리 |
| **Event Sourcing** | 이벤트가 진실의 원천, 상태는 파생 |
| **Outbox Pattern** | 분산 트랜잭션을 단일 DB 트랜잭션으로 우회 |
| **Outbox Transport** | Kafka든 Redis+Celery든 갈아끼울 수 있음 |

---

## 9. 레퍼런스

- [Cosmic Python — Repository Pattern](https://www.cosmicpython.com/book/chapter_02_repository.html)
- [cosmicpython/code (GitHub)](https://github.com/cosmicpython/code) — MADE.com 프로덕션 기반
- [Dagster Storage Abstraction](https://github.com/dagster-io/dagster) — EventLogStorage / RunStorage
- [SAP S/4HANA NSDM](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/sap-s-4hana-inventory-management-tables-new-simplified-data-model-nsdm/ba-p/13497469)
- [frappe/erpnext (GitHub)](https://github.com/frappe/erpnext)
- [Microservices.io — Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Martin Fowler — CQRS](https://martinfowler.com/bliki/CQRS.html)
- [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Alistair Cockburn — Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [Gary Bernhardt — Boundaries (Functional Core, Imperative Shell)](https://www.destroyallsoftware.com/talks/boundaries)
- [Debezium — Change Data Capture](https://debezium.io/)
