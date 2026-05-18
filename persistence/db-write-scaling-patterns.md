# DB Write 스케일링 패턴 — 업계 실사례 학습 노트

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 고부하 환경에서 DB write 병목/커넥션 풀 포화를 어떻게 푸는가? 업계는 실제로 무엇을 쓰는가? |
| 날짜 | 2026-04-23 (최초) |
| 관련 ADR | ADR-011 (DB 레이어), ADR-012 (Worker 런타임), ADR-014 (State-Oriented Pivot) |

---

## 1. "비동기" 의 두 가지 의미 — 혼동 방지

DB 맥락에서 "비동기" 라고 하면 완전히 다른 두 개념이 섞여 쓰임. 혼동하면 설계 판단 엉킨다.

### 의미 A — **Async I/O 드라이버** (asyncpg, psycopg3 async, Tokio, Node pg)
- 같은 SQL 의미. 앱이 `await` 로 결과 기다림
- I/O wait 동안 다른 coroutine 진행 가능 — **단일 프로세스가 더 많은 동시 요청 처리**
- **DB 부하 자체는 sync 와 동일**. 커넥션 수 동일. 쿼리 지연 동일
- 벤치: asyncpg 가 psycopg2 대비 high concurrency 에서 **2-3배 throughput** — I/O overlap 효과
- "fire-and-forget" 아님, "queue 로 미룸" 아님

### 의미 B — **Queue-backed async** (Celery, RabbitMQ, FOQS, Instagram/Facebook 패턴)
- 앱이 큐에 task 넣고 **즉시 응답**
- 별도 worker 가 큐 drain → DB write
- 호출자와 실제 DB write 시점 **분리**
- **Eventual consistency** — durability 저하 수용

**중요**: 의미 A 는 "어떻게 쓰느냐" 의 drop-in replacement. 의미 B 는 "언제 쓰느냐" 를 바꾸는 아키텍처 변경.

---

## 2. Connection Pool 문제의 본질

### 왜 발생

PostgreSQL 은 **프로세스 기반 커넥션**. 1 커넥션 = 1 백엔드 프로세스 = 1-3MB 메모리 + context switch. 10,000 커넥션이면 연결 오버헤드만 20GB.

### 실측 한계 (교차 확인)

- **최적 active connection ≈ (core × 2) + disk count** (EDB, PostgreSQL wiki)
- OLTP throughput peak: **200-300 active connection** 근처
- `max_connections` 권장: core 당 **2-3개**
- 실제 write 한계: 고사양 **~1,875 writes/sec**, 일반 SSD **1,000-1,200 writes/sec**
- **Idle 커넥션도 자원 먹음** (스냅샷 관리, shared memory) — 많으면 active 성능도 저하

### 수직 확장의 벽

`synchronous_commit=on` 이면 모든 커밋이 **WAL fsync** 를 기다려야 한다. 튜닝으로 못 뚫는 아키텍처 제약. 초당 수만 건 durable write 는 **샤딩/구조 변경 필수**.

---

## 3. 각 전략이 "무엇을" 해결하나 — 메커니즘 분류

| 전략 | (a) write rate 감소 | (b) peak 분산 | (c) 커넥션 재사용 | (d) 문제 이동만 |
|---|:---:|:---:|:---:|:---:|
| Async driver (asyncpg) | | | ✓ (같은 pool 더 많은 coroutine) | |
| asyncio.Semaphore | | ✓ backpressure | | |
| In-process queue + batching | ✓ (N→1) | ✓ | ✓ | |
| Bulk INSERT / COPY | ✓ (N→1) | | | |
| Fire-and-forget | | | | ✓ **문제 숨김만** |
| Write-behind cache | ✓ conflation | ✓ | | |
| **pgbouncer / PgCat / PgBouncer-compat** | | | **✓✓✓ 10K→200 커넥션** | |
| RDS Proxy / Neon pooler | | | ✓✓ | |
| Kafka / Redis Streams | | ✓✓ | ✓ (consumer 수 제한) | ✓ (sink worker 로 이동) |
| Outbox / CDC | | ✓ | | ✓ (원본 write 는 앱→PG 동일) |

### 핵심 인사이트

**"Kafka 를 넣으면 connection pool 문제가 풀린다"** 는 **consumer 가 DB 쓸 때 거기서 재발** 한다는 뜻. Kafka 자체가 write 를 없애지 않음 — 쓰는 주체를 **"수많은 앱 인스턴스" → "소수 sink worker"** 로 옮겨 **"예측 가능"** 하게 만듦.

**Connection pool 에 가장 직접적 해결은 pooler 계층** (pgbouncer 등). 이건 논쟁 없이 업계 표준.

---

## 4. 업계 실사례 (규모별)

### 4.1 Discord — Kafka 없이 ScyllaDB direct (trillion msg/day)

- Cassandra → ScyllaDB 마이그레이션 (2023)
- **Rust "data services" 레이어** 에서 request coalescing — 동일 메시지 중복 요청 1회로 병합
- 성과: latency 200ms → 5ms, 노드 수 177 → 72
- **왜 Kafka 없이**: DB 자체가 write-optimized (shard-per-core). In-app coalescing 으로 peak 완화
- **분류**: DB 선택 + 앱 내 merging. 의미 A/B 둘 다 아님

출처:
- [How Discord Stores Trillions of Messages](https://discord.com/blog/how-discord-stores-trillions-of-messages)
- [How Discord Migrated Trillions of Messages from Cassandra to ScyllaDB](https://www.scylladb.com/tech-talk/how-discord-migrated-trillions-of-messages-from-cassandra-to-scylladb/)

### 4.2 Shopify — MySQL Pod 샤딩 (BFCM 2025 피크 7.6M writes/sec)

- BFCM 2025 순간 피크: **489M RPM, 7.6M writes/sec, 14.8조 쿼리/day, 1.75조 writes/day**
- **Pod 단위 MySQL 샤딩**. Pod Balancer 가 binlog 기반으로 shop 을 pod 간 재배치
- 9개월·5차례 scale test 로 150% 부하 검증
- **분류**: DB 아키텍처 (샤딩). 의미 A/B 아님

출처:
- [How we prepare Shopify for BFCM (2025)](https://shopify.engineering/bfcm-readiness-2025)
- [A Pods Architecture To Allow Shopify To Scale](https://shopify.engineering/a-pods-architecture-to-allow-shopify-to-scale)

### 4.3 Stack Overflow — SQL Server 모놀리스

- SQL Server 2대 (master + replica), AlwaysOn Availability Groups
- **SQL CPU 평균 5-10%** — 하드웨어가 여유
- Dapper micro-ORM, stored procedure 1개
- **분류**: 수직 확장 + 설계 단순성. 의미 A/B 아님
- 교훈: "**단일 RDBMS 도 올바른 설계면 수백만 사용자 서비스 가능**"

출처:
- [Stack Overflow: The Architecture 2016 — Nick Craver](https://nickcraver.com/blog/2016/02/17/stack-overflow-the-architecture-2016-edition/)

### 4.4 Instagram — Celery + RabbitMQ (의미 B, 특정 작업만)

- **200 Python worker** 가 task queue 소비
- Queue 에 넣는 작업:
  - 이메일, 푸시 알림
  - 미디어 파일 transcoding
  - Feed fanout (Gearman)
  - likes/comments **카운터 업데이트**
  - 사용자 활동 analytics 로깅
  - 프로필 통계 재계산
- Queue 에 **안 넣는** 작업:
  - 실제 포스트 업로드 기록 (DB 직접)
  - 팔로우 관계 변경 (DB 직접)
  - 로그인/인증
- **분류**: 의미 B (queue-backed) — **단, non-critical 작업 한정**

출처:
- [Instagram Feed & Queues](https://medium.com/@abutallah.n/how-instagram-loads-your-feed-so-smoothly-the-secret-power-of-queues-f2b2e7273f38)
- [Instagram Engineering — What Powers Instagram](https://instagram-engineering.com/what-powers-instagram-hundreds-of-instances-dozens-of-technologies-adf2e22da2ad)

### 4.5 Facebook FOQS — 샤딩된 MySQL 기반 priority queue

- **Trillion items** 처리하는 분산 priority queue
- MySQL 위에 구축, horizontally scalable, multi-tenant
- Prefetch Buffer 로 dequeue 는 메모리에서 서비스
- Multi-region binlog replication (disaster-ready)
- Delay tolerance per job — job 별 허용 지연 명시
- **분류**: 의미 B (queue-backed) — Facebook 의 내부 async 작업 전반

출처:
- [FOQS: Scaling a distributed priority queue — Meta](https://engineering.fb.com/2021/02/22/production-engineering/foqs-scaling-a-distributed-priority-queue/)
- [FOQS disaster-ready](https://engineering.fb.com/2022/01/18/production-engineering/foqs-disaster-ready/)
- [Meta Async Computing — Overview](https://engineering.fb.com/2023/01/31/production-engineering/meta-asynchronous-computing/)

### 4.6 Netflix Keystone — Kafka 2단 (2조 msg/day)

- 2조 msg/day, 3PB in / 7PB out, 100+ Kafka 클러스터 + Flink
- **Fronting Kafka → Consumer Kafka** 2단 구조 (bulkhead isolation)
- at-least-once, timestamp-based sequencing, drop rate < 0.01%
- 철학: **"사용자 서비스 저하보다 event drop 이 낫다"**
- **분류**: 의미 B 의 극한 버전 — 모든 이벤트가 Kafka 경유

출처:
- [Keystone Real-time Stream Processing Platform — Netflix](https://netflixtechblog.com/keystone-real-time-stream-processing-platform-a3ee651812a)

### 4.7 Stripe — MongoDB 기반 DocDB

- 5M queries/sec, 99.999% 가용성
- 자체 DocDB (MongoDB 기반), 수천 샤드
- 2023 한 해에 petabyte 규모 shard 마이그레이션
- **분류**: 수평 샤딩. critical write 는 sync direct

출처:
- [How Stripe Scaled to 5M Database Queries Per Second — ByteByteGo](https://blog.bytebytego.com/p/how-stripe-scaled-to-5-million-database)

### 4.8 Figma — PostgreSQL + 자체 DBProxy

- 9개월에 걸쳐 horizontal sharding. CockroachDB/TiDB/Spanner/Vitess 검토 후 **직접 구축**
- "Logical shard" (앱 레이어) vs "Physical shard" (PG 레이어) 분리
- DBProxy 가 SQL 을 AST 로 파싱해 각 shard 로 라우팅
- **Shadow readiness framework** 로 shard key 후보별 트래픽 분포 예측
- **분류**: DB 아키텍처 (샤딩 + 프록시 레이어)

출처:
- [How Figma's Databases Team Lived to Tell the Scale](https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/)

### 4.9 Uber — 다층 전략

- **Schemaless** (MySQL 기반 append-only) — 수천 클러스터, 수십 PB
- **Kafka uForwarder** — 1000+ downstream service 에 trillion msg/day push proxy
- Federated cluster (단일 monolith 금지, ~150 노드 클러스터 여러 개)
- **Tiered storage** (hot SSD + cold remote)
- **분류**: 의미 B + 샤딩 + 프록시 복합

출처:
- [Designing Schemaless — Uber](https://www.uber.com/us/en/blog/schemaless-part-one-mysql-datastore/)
- [uForwarder: Kafka Async Queuing Proxy](https://www.uber.com/us/en/blog/introducing-ufowarder/)

---

## 5. 각 회사의 **선택 축** 분류

| 회사 | DB 선택 | 샤딩 | 커넥션 풀러 | 의미 A (async driver) | 의미 B (queue-backed) |
|------|---------|------|-----------|----------------------|---------------------|
| Discord | ScyllaDB | per-core | — | — | **없음** (critical path) |
| Shopify | MySQL | **Pod 단위** | 있음 | — | — |
| Stack Overflow | SQL Server | 없음 (2대) | 있음 | — | — |
| Instagram | PostgreSQL(초기), Cassandra | 있음 | 있음 | — | **Celery+RabbitMQ** (non-critical) |
| Facebook | MySQL | 있음 | 있음 | — | **FOQS** (Facebook 내부 async 작업) |
| Netflix | Cassandra + Kafka | 있음 | 있음 | — | **Kafka Keystone** (모든 event) |
| Stripe | 자체 DocDB | 수천 샤드 | 있음 | — | — (critical path sync) |
| Figma | PostgreSQL + DBProxy | 있음 | 있음 | — | — |
| Uber | Schemaless + Kafka | 있음 | 있음 | — | **Kafka uForwarder** |

### 핵심 패턴 요약

1. **Critical writes (결제/트랜잭션/실시간 메시지) 는 sync direct**. Discord, Stripe, Shopify 결제 경로 공통
2. **Non-critical writes (알림/카운터/분석) 는 queue-backed OK**. Instagram, Facebook 패턴
3. **Netflix 처럼 "모든 이벤트 Kafka" 는 trillion/day 규모 + 특수 요구 (replay, 다수 consumer) 때**
4. **샤딩 없이 100만 사용자**: Stack Overflow (SQL Server), Discord (ScyllaDB), Stripe (초기 PG)
5. **pooler (pgbouncer/RDS Proxy/Supavisor) 는 거의 모든 곳이 씀** — 논쟁 없는 기본값

---

## 6. 규모별 결정 매트릭스

| 초당 write | 주류 관행 | 대표 사례 |
|-----------|----------|-----------|
| **\<1K** | 앱 내 pool (asyncpg/pgx). pgbouncer 조차 오버킬 | 대부분 스타트업 MVP |
| **1K-10K** | **pgbouncer transaction mode 필수**. pool 문제 첫 등장 구간 | Markaicode: 8K→50K RPS 달성 사례 |
| **10K-100K** | pgbouncer + read replica + **monthly partitioning**. Spike 있으면 Redis Streams/SQS buffering | Shopify 이전 규모 |
| **100K+** | **샤딩 불가피**. Vitess(MySQL), Figma DBProxy(PG). Kafka 가 주 ingress buffer | Shopify, Figma, GitHub |
| **1M+** | 다층 분산 DB (Scylla/Cassandra/Spanner 또는 자체) + Kafka 2단 + federated cluster | Discord, Uber, Netflix |

---

## 7. 오해 교정 (자주 하는 실수)

### 오해 1: "Kafka 넣으면 pool 문제 해결"

**틀림.** Consumer 가 DB 에 쓰는 순간 거기서 재발. 단 consumer 수를 개발자가 통제 가능해져 **예측 가능** 해짐. Kafka 는 write 부하를 **시간적으로 분산 + 주체 제한** 하지 **없애지 않음**.

### 오해 2: "Async driver 쓰면 DB 부하 줄어든다"

**틀림.** 의미 A 는 I/O overlap 이득. 같은 pool 을 더 많은 coroutine 이 공유. DB 는 모름. **커넥션 수 동일, 쿼리 속도 동일**.

### 오해 3: "대형 서비스는 모두 Kafka 쓴다"

**틀림.** Discord (trillion msg/day) 는 Kafka 없이 ScyllaDB direct. Shopify BFCM (7.6M writes/sec peak) 도 Kafka 가 핵심 아니고 **MySQL pod 샤딩**. Kafka 는 **"모든 이벤트 재생/다수 consumer 분배 필요"** 할 때 정답.

### 오해 4: "Async write 는 fire-and-forget 의미"

**틀림.** 의미 A 는 durability 동일. 의미 B 는 **eventual consistency 가 수용 가능한 작업 한정** — Instagram 도 로그인/포스트 업로드는 sync direct.

### 오해 5: "커넥션 풀러는 선택사항"

**틀림.** pgbouncer/PgCat/RDS Proxy 는 **업계 표준 기본값**. 논쟁 없음. Single-threaded pgbouncer 한계(25+ 클라이언트) 에서 multi-threaded PgCat/Supavisor 등장.

---

## 8. AgentPit 맥락 교훈

AgentPit 예상 규모 (10만 alive / 1-2만 active / 턴 기반 30초 주기):
- 초당 agent activity: ~333-666
- 초당 DB write: **peak 1K-3K, 평시 이하**
- 업계 매트릭스: **"\<1K ~ 1K-10K 하단"**

이 구간의 업계 권장:
- **asyncpg async driver (의미 A)**: ✓ 이미 쓰고 있음
- **asyncio.Semaphore backpressure**: ✓ ADR-012 에 있음
- **pgbouncer**: 아직 이른 단계 (worker pod 10개 × pool 8 = 80 커넥션, PG max 200 여유)
- **Kafka / 의미 B queue**: **과설계**. Discord 참고 — "DB 제대로 쓰면 큐 불필요"

**Phase 1 에 의미 B 가 적합한 작업 후보** (Phase 2 고민 시):
- **관찰 이벤트** (preview 실패 로그): eventual OK, Instagram 분석 로그와 유사
- **evaluator 배치 집계**: 세션 종료 후 처리, 이미 Parquet→S3 로 계획됨

**Critical writes 는 영원히 sync direct**:
- trade 성사 (LLM tool 응답에 결과 들어감)
- message 발송 (게임 턴)
- offer 생성

---

## 9. 설계 원칙 (정리)

1. **"비동기" 두 의미 구분**. 설계 논의 시 어느 쪽인지 명시.
2. **Critical writes 는 sync direct**. 업계 예외 없음 (Discord, Stripe, Shopify 결제).
3. **Non-critical writes 는 queue-backed 가능**. 단 "eventual OK" 만.
4. **Connection pool 의 정면 해결은 pooler 계층** (pgbouncer 등). 큐는 peak 분산 + 주체 제한.
5. **Kafka 는 "replay + 다수 consumer + cross-region"** 일 때 정답. Write scaling 자체엔 부차.
6. **샤딩은 수평 확장의 진짜 답**. 큐/비동기로 못 뚫는 수직 한계는 결국 샤딩.
7. **측정 먼저, 도입 나중**. "10만 가정" 으로 미리 Kafka 넣지 말 것 — Discord/Figma 가 실증.

---

## 10. 참고 — Source of Truth

**수치 근거**:
- [Number Of Database Connections — PostgreSQL wiki](https://wiki.postgresql.org/wiki/Number_Of_Database_Connections)
- [Why you should use Connection Pooling — EDB](https://www.enterprisedb.com/postgres-tutorials/why-you-should-use-connection-pooling-when-setting-maxconnections-postgres)
- [PostgreSQL Write Performance: What the Benchmarks Won't Tell You — dev.to](https://dev.to/haikasatryan/postgresql-write-performance-what-the-benchmarks-wont-tell-you-mm7)
- [Why WAL and Write Amplification Become the Hidden Scaling Limit — c-sharpcorner](https://www.c-sharpcorner.com/article/why-wal-and-write-amplification-become-the-hidden-scaling-limit-in-postgresql/)

**드라이버 벤치마크**:
- [Psycopg 3 vs Asyncpg — Fernando Arteaga](https://fernandoarteaga.dev/blog/psycopg-vs-asyncpg/)
- [Psycopg2 vs Psycopg3 Performance Benchmark — Tiger Data](https://www.tigerdata.com/blog/psycopg2-vs-psycopg3-performance-benchmark)

**커넥션 풀러 비교**:
- [PgBouncer vs Pgcat vs Odyssey on VPS in 2025 — Onidel](https://onidel.com/blog/postgresql-proxy-comparison-2025)
- [Benchmarking PostgreSQL connection poolers — Tembo](https://legacy.tembo.io/blog/postgres-connection-poolers/)
- [PgBouncer is useful, important, and fraught with peril — JP Camara](https://jpcamara.com/2023/04/12/pgbouncer-is-useful.html)

**메시지 브로커 비교**:
- [Real-Time Event Streaming: Kafka vs Redis Streams vs NATS in 2026 — DEV](https://dev.to/young_gao/real-time-event-streaming-kafka-vs-redis-streams-vs-nats-in-2026-34o1)
- [NATS and Kafka Compared — Synadia](https://www.synadia.com/blog/nats-and-kafka-compared)

**업계 사례** (위 §4 각 항목에 이미 출처 링크 있음)
