# DB 저장소 분리 전략: RDBMS vs NoSQL

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 어떤 데이터를 RDBMS에 두고 어떤 데이터를 NoSQL에 두는가? |

---

## 1. 판단 기준

| 기준 | RDBMS | NoSQL |
|------|-------|-------|
| 관계 | FK/join 필요 | 독립적, 다른 테이블 참조 불필요 |
| 연산 | 읽기 + 쓰기 + 수정 혼합 | 쓰기 위주 or 읽기 위주 (한쪽) |
| 일관성 | 트랜잭션으로 여러 테이블 묶어야 함 | eventual consistency OK |
| 수정 패턴 | UPDATE/DELETE 빈번 | append-only, 수정 안 함 |
| 스키마 | 고정, 변경 시 마이그레이션 | 유연, 필드 추가 자유 |
| 쿼리 | 임의 SQL, JOIN, 집계 강력 | key-based 조회, 일부 집계 |

---

## 2. 핵심 원칙

- **join 성이 짙은 것 + 트랜잭션으로 묶어야 하는 것** → RDBMS
- **join 필요 없고 + 오직 조회나 쓰기만 하는 것** → NoSQL

이 분리는 데이터의 본질에 따라 결정된다. 같은 시스템 안에서도 데이터 종류마다 다른 저장소를 쓸 수 있다 (polyglot persistence).

---

## 3. 일반적인 데이터 분류

### RDBMS에 적합 (PostgreSQL, MySQL)

| 데이터 종류 | 특징 |
|-----------|------|
| 사용자/계정 | 다른 테이블이 FK로 참조, 수정 빈번 |
| 주문/결제 | 트랜잭션 원자성 필수 (재고 차감 + 결제 + 주문 생성) |
| 재고/잔고 | 동시 수정 충돌 처리 필요 (`SELECT FOR UPDATE`) |
| 관계 (친구, 팔로우) | JOIN 쿼리 빈번 |
| 권한/RBAC | 트랜잭션 안에서 검증 |

### NoSQL에 적합

#### Document store (MongoDB)
| 데이터 종류 | 특징 |
|-----------|------|
| 이벤트 로그 | append-only, 수정 없음 |
| 활동 피드 | 시간순 조회, 사용자별 필터 |
| 로그/감사 추적 | 쓰기 폭주, 분석용 읽기 |
| CMS 콘텐츠 | 스키마 유연 |

#### Key-value store (Redis, DynamoDB)
| 데이터 종류 | 특징 |
|-----------|------|
| 세션 | 빠른 조회, TTL |
| 캐시 | 만료 가능, 휘발성 OK |
| 카운터/leaderboard | 원자적 INCR, sorted set |
| 큐 | LPUSH/BRPOP |

#### Wide-column (Cassandra, ScyllaDB)
| 데이터 종류 | 특징 |
|-----------|------|
| 시계열 메트릭 | 시간 + 키 조합 쓰기 폭주 |
| IoT 센서 데이터 | 분산 쓰기 확장 |

#### Search engine (Elasticsearch, OpenSearch)
| 데이터 종류 | 특징 |
|-----------|------|
| 전문 검색 | 형태소 분석, 역인덱스 |
| 로그 분석 | Kibana 대시보드 |

---

## 4. Polyglot Persistence — 한 시스템 안 여러 DB

대부분의 현대 백엔드는 단일 DB가 아니다. 예시:

```
[E-commerce 백엔드]
  PostgreSQL  → 사용자, 주문, 결제, 재고 (트랜잭션 핵심)
  MongoDB     → 상품 카탈로그 (스키마 유연), 리뷰
  Redis       → 세션, 장바구니 (TTL), 인기 상품 캐시
  Elasticsearch → 상품 검색
  S3          → 이미지, 첨부 파일
  Kafka       → 이벤트 스트림 (다른 마이크로서비스로)
```

각 DB는 자기가 잘하는 것만 한다. **억지로 한 DB에 다 넣지 마라.**

### Polyglot의 비용

- 운영 복잡도 증가 (백업/모니터링/장애 대응)
- 데이터 일관성 어려움 (분산 트랜잭션)
- 학습 곡선 (팀이 여러 DB를 이해해야 함)

→ **시작은 단일 RDBMS, 명확한 이유가 생기면 분리**가 안전.

---

## 5. 분리 트리거 — 언제 NoSQL 추가하나?

PostgreSQL 하나로 시작했다가 NoSQL을 추가하는 시점:

| 트리거 | 추가할 NoSQL |
|--------|------------|
| 이벤트 로그가 PostgreSQL을 잠식 (수억 row) | MongoDB, Cassandra |
| 검색 쿼리가 LIKE/ILIKE로 느려짐 | Elasticsearch |
| 세션을 PostgreSQL에 두니 connection pool 고갈 | Redis |
| 캐시가 필요해짐 | Redis, Memcached |
| 시계열 메트릭이 PostgreSQL을 잠식 | TimescaleDB, InfluxDB, Prometheus |
| 큐 처리량이 PostgreSQL 한계 | Redis, RabbitMQ, Kafka |

조기 최적화 X. **실제 병목이 보일 때 분리**.

---

## 6. 일관성 처리

여러 DB로 분리하면 자연스럽게 분산 트랜잭션 문제가 발생한다.

### 단일 DB일 때
```python
BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 100 WHERE id = 1
  UPDATE accounts SET balance = balance + 100 WHERE id = 2
COMMIT
```
원자성 보장. 끝.

### 여러 DB일 때
```python
postgres.update("UPDATE accounts ...")  # ① 성공
mongodb.insert({"event": "transfer"})    # ② 실패 → ①은 이미 commit됨
```
원자성 깨짐. 해결책:

| 패턴 | 설명 | 적합 |
|------|------|------|
| **Outbox Pattern** | 이벤트를 같은 DB의 outbox 테이블에 적고, relay가 외부로 옮김 | 대부분의 경우 |
| **Saga** | 보상 트랜잭션으로 부분 성공 롤백 | 마이크로서비스 사이 |
| **Single source of truth** | 한 DB가 진실, 나머지는 derived view | 분석/검색 위주 |
| **2PC / XA** | 두 DB가 분산 트랜잭션 프로토콜 참여 | 거의 사용 X (성능/복잡도) |

상세는 [`cqrs-outbox-and-repository-pattern.md`](cqrs-outbox-and-repository-pattern.md) 참고.

---

## 7. 안티패턴

### 7.1 RDBMS에 JSON 떡칠
PostgreSQL JSONB는 강력하지만, **모든 데이터를 JSONB 컬럼 하나에 넣으면** 관계형의 장점을 다 잃는다. 정말 동적 스키마가 필요한 경우만 사용.

### 7.2 NoSQL을 트랜잭션처럼 사용
MongoDB도 트랜잭션을 지원하지만, **여러 도큐먼트에 걸친 트랜잭션은 성능 페널티**가 크다. 자주 함께 수정되는 데이터는 같은 도큐먼트에 임베딩.

### 7.3 캐시를 source of truth처럼 사용
Redis는 휘발성 (RDB/AOF로 어느 정도 지속화 가능하지만 RDBMS 수준 X). 진실은 RDBMS에 두고 Redis는 캐시로만.

### 7.4 검색 엔진에 트랜잭션 데이터 저장
Elasticsearch는 검색에 최적화되어 있고, **near-realtime** (수 초 지연). 잔고 같은 즉시 일관성 필요한 데이터는 RDBMS에.

### 7.5 일찍 분리
프로덕션 트래픽 보기도 전에 5개 DB로 분리하면 운영 비용만 늘어나고 가치 없음. **실제 병목 보고 분리.**

---

## 8. 결론

> **시작은 단일 RDBMS (PostgreSQL). 명확한 분리 이유가 생기면 polyglot으로 확장.**

분리 시 원칙:
- join + 트랜잭션 → RDBMS
- append-only + 분석 → NoSQL document/wide-column
- 빠른 조회 + TTL → key-value
- 전문 검색 → search engine
- 일관성은 outbox로 해결

---

## References

- [Martin Kleppmann — Designing Data-Intensive Applications](https://dataintensive.net/) — Chapter 3 (Storage), Chapter 5 (Replication), Chapter 9 (Consistency)
- [Microservices.io — Database per Service](https://microservices.io/patterns/data/database-per-service.html)
- [PostgreSQL JSONB documentation](https://www.postgresql.org/docs/current/datatype-json.html)
- [MongoDB Schema Design Best Practices](https://www.mongodb.com/developer/products/mongodb/mongodb-schema-design-best-practices/)
- [Redis Persistence](https://redis.io/docs/management/persistence/)
- [Polyglot Persistence — Martin Fowler](https://martinfowler.com/bliki/PolyglotPersistence.html)
