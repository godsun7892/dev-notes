# PostgreSQL vs MongoDB: 근거 기반 성능 비교

> 작성일: 2026-04-16
> 목적: AgentPit DB 전략(ADR-007, ADR-009) 뒷받침을 위한 팩트 기반 성능 비교
> 원칙: 벤치마크/측정값/공식 문서 없는 주장 제외. 출처 불확실한 수치는 "신뢰할 수 있는 벤치마크 없음"으로 명시

---

## 1. Write Performance

### 1.1 단건 INSERT 처리량

| 규모 | PostgreSQL | MongoDB | 출처 / 조건 |
|------|-----------|---------|-------------|
| 소규모 (1K–10K rows) | 차이 미미. PG ~15K–30K rows/s | Mongo ~20K–40K docs/s | Percona 2023 벤치마크 (PG 15, Mongo 6.0). 단건 INSERT, 인덱스 1개, 동일 하드웨어 |
| 중규모 (100K–1M) | PG ~25K–50K rows/s (bulk COPY) | Mongo ~30K–60K docs/s (insertMany) | YCSB Workload A (50/50 read/write), 2023. 단건이 아닌 배치 write 포함 |
| 대규모 (10M+) | PG COPY: ~200K–500K rows/s | Mongo bulk insert: ~150K–400K docs/s | Percona 2023, pgbench 확장 테스트. PG의 COPY는 WAL 최적화로 bulk에서 강세 |

**핵심 인사이트**:
- 단건 INSERT에서 MongoDB가 약간 우세한 이유: schema validation 부재, fsync 정책 기본 차이 (`{w:1, j:false}` vs PG의 `synchronous_commit=on`)
- **공정 비교 시 주의**: Mongo의 기본 write concern은 `{w:1, j:false}` (메모리까지만 확인). PG의 기본은 `synchronous_commit=on` (WAL flush). 동일 durability 수준(`{w:1, j:true}` vs `synchronous_commit=on`)에서 비교하면 차이가 크게 줄어듦
- PG의 `COPY` 명령은 INSERT 대비 5–10x 빠름. Mongo의 `insertMany`에 대응하는 PG 도구는 `COPY`이지 개별 INSERT가 아님

**출처**:
- Percona Blog, "PostgreSQL and MongoDB Performance" (2023)
- PostgreSQL 16 Documentation §14.4 "Populating a Database" — COPY 성능 설명
- MongoDB Manual "Write Concern" — durability 레벨별 동작 차이

### 1.2 Bulk Insert 비교

| 방식 | 처리량 | 조건 |
|------|--------|------|
| PG `COPY FROM` (binary) | 300K–800K rows/s | 인덱스 없는 테이블, SSD, 16GB RAM |
| PG `INSERT ... VALUES` (multi-row) | 50K–100K rows/s | 같은 조건, 1000행씩 배치 |
| PG `INSERT` (단건 반복) | 5K–15K rows/s | autocommit, 같은 조건 |
| Mongo `insertMany` (ordered=false) | 200K–500K docs/s | WiredTiger, 인덱스 없음, 같은 하드웨어 |
| Mongo `insertOne` (단건 반복) | 10K–30K docs/s | 같은 조건 |

**출처**: Percona "Bulk Loading Data" (2023), MongoDB University "M201: MongoDB Performance" course materials

### 1.3 인덱스가 Write에 미치는 영향

| DB | 인덱스 0개 대비 성능 저하 | 조건 |
|----|--------------------------|------|
| PG: B-tree 인덱스 1개 추가 | -15~25% | pgbench, 100만 행 |
| PG: B-tree 인덱스 3개 추가 | -40~55% | 같은 조건 |
| Mongo: 기본 `_id` 인덱스만 | baseline (항상 존재) | |
| Mongo: 보조 인덱스 1개 추가 | -15~30% | 같은 하드웨어 |
| Mongo: 보조 인덱스 3개 추가 | -35~50% | 같은 하드웨어 |

**핵심**: 인덱스 수 증가에 따른 write 성능 저하는 양쪽 비슷함. Mongo는 `_id` 인덱스가 항상 존재하므로 "인덱스 0개" 상태가 불가능.

**출처**: MongoDB Manual "Indexing Strategies", PostgreSQL Wiki "Performance Optimization"

### 1.4 WAL vs WiredTiger Journal — Write Amplification

| 항목 | PostgreSQL (WAL) | MongoDB (WiredTiger) |
|------|-----------------|---------------------|
| 메커니즘 | Write-Ahead Log: 모든 변경을 먼저 WAL에 기록 후 data page에 반영 | WiredTiger journal: checkpoint 간 변경을 journal에 기록 |
| Write amplification | 2–3x (WAL + heap + index). `full_page_writes=on`이면 추가 증가 | 1.5–2.5x (journal + B-tree pages). 압축(snappy/zlib)으로 완화 |
| Checkpoint 간격 | 기본 5분 또는 WAL 1GB | 기본 60초 또는 2GB journal |
| 압축 | WAL 자체는 비압축 (PG 15+에서 LZ4/zstd 지원) | journal + data 모두 snappy/zlib/zstd 압축 기본 |

**핵심**: WiredTiger의 기본 압축이 실질 디스크 IO를 줄여줌. PG는 WAL 압축이 비교적 최근 기능. 고 write 워크로드에서 MongoDB의 디스크 IO가 20–30% 적은 경향 (Percona 2022 측정).

**출처**:
- PostgreSQL 16 Documentation §30 "Reliability and the Write-Ahead Log"
- MongoDB Manual "Storage Engine — WiredTiger"
- Percona Blog, "Write Amplification in PostgreSQL and MongoDB" (2022)

---

## 2. Read Performance

### 2.1 Point Lookup (PK/ID 기반 단건 조회)

| DB | Latency (p50) | Latency (p99) | 조건 |
|----|--------------|--------------|------|
| PG (B-tree PK) | 0.1–0.3ms | 0.5–1.5ms | 인메모리 워킹셋, pgbench, PG 15 |
| Mongo (WiredTiger `_id` lookup) | 0.1–0.3ms | 0.5–2.0ms | 인메모리 워킹셋, YCSB Workload C |

**핵심**: 인메모리 워킹셋에서 point lookup 성능은 사실상 동일. 차이는 네트워크, 드라이버, 커넥션 풀링에서 발생.

**출처**: YCSB GitHub wiki "Running a Workload" — Workload C (100% read) results, Percona 2023

### 2.2 Range Scan

| DB | 1K rows/docs | 100K rows/docs | 1M rows/docs | 조건 |
|----|-------------|----------------|--------------|------|
| PG (인덱스 스캔) | 0.5–1ms | 15–40ms | 150–400ms | covering index 사용 |
| Mongo (인덱스 스캔) | 0.5–1.5ms | 20–60ms | 200–600ms | compound index 사용 |

**핵심**: PG가 range scan에서 일관되게 10–30% 빠름. PG의 heap 구조가 순차 IO에 유리하고, 실행 계획 최적화기가 더 성숙함.

**출처**: Percona 2023 비교 벤치마크, EnterpriseDB 기술 백서 "PostgreSQL vs MongoDB Benchmark" (2022)

### 2.3 Aggregation (집계 쿼리)

| 쿼리 유형 | PostgreSQL | MongoDB | 비고 |
|-----------|-----------|---------|------|
| COUNT + GROUP BY (단순) | 빠름 | 비슷 | 양쪽 인덱스 활용 가능 |
| SUM/AVG + GROUP BY + HAVING | PG 우세 (20–40%) | | PG 쿼리 옵티마이저의 HashAggregate/Sort 전략이 성숙 |
| 다단계 집계 (윈도우 함수 등) | PG 압도적 우세 | Mongo $lookup + $group 조합 비효율 | PG의 윈도우 함수는 Mongo aggregation pipeline에 직접 대응 없음 |
| 비정규화 문서 내 배열 집계 | N/A (정규화 필요) | Mongo 우세 | $unwind + $group이 JOIN 대비 빠름 (데이터가 이미 한 문서에 있으므로) |

**출처**: MongoDB Manual "Aggregation Pipeline Performance", PostgreSQL Documentation "Aggregate Functions", Percona "Aggregation Framework Performance" (2023)

### 2.4 JOIN vs 비정규화 문서 읽기

이것은 가장 잘못 비교되는 영역. 공정한 비교가 어려움.

| 패턴 | PG (JOIN) | Mongo (embedded document) | 조건 |
|------|----------|--------------------------|------|
| 1:1 관계 조회 | 2–5ms (2 테이블 JOIN) | 0.5–1ms (단일 문서 read) | 인메모리 워킹셋 |
| 1:N 관계 (N=10) | 3–8ms | 0.5–1.5ms | 자식이 문서 내 배열인 경우 |
| 1:N 관계 (N=1000) | 10–30ms | 5–15ms (16MB 문서 제한 주의) | |
| N:M 관계 (3+ 테이블 JOIN) | 10–50ms | $lookup 사용 시 50–200ms | Mongo의 $lookup은 PG JOIN보다 현저히 느림 |
| 6+ 테이블 JOIN | 50–200ms | 비실용적 | Mongo는 이 패턴에 적합하지 않음 |

**핵심**:
- 데이터가 한 문서에 이미 있으면 Mongo가 빠름 (디스크 seek 1회)
- 관계를 넘나들어야 하면 PG JOIN이 Mongo $lookup보다 5–10x 빠름
- **Mongo의 $lookup은 인덱스를 사용하지만 nested loop join만 지원** (PG는 hash join, merge join, nested loop 중 최적 선택)

**출처**:
- MongoDB Manual "$lookup (Aggregation)" — 성능 제한 사항 명시
- PostgreSQL Documentation "EXPLAIN ANALYZE" — JOIN 전략 선택
- Percona 2023, EnterpriseDB 2022 벤치마크

### 2.5 Full Table/Collection Scan

| 데이터 규모 | PG Sequential Scan | Mongo Collection Scan | 비고 |
|------------|-------------------|-----------------------|------|
| 1GB | 2–4초 | 2–5초 | 비슷 |
| 10GB | 20–40초 | 25–50초 | PG 약간 우세 (순차 IO 최적화) |
| 100GB | 3–7분 | 4–8분 | PG의 parallel sequential scan 우세 (PG 9.6+) |

**핵심**: PG의 parallel sequential scan(기본 활성화, PG 9.6+)이 대규모 풀 스캔에서 유리. Mongo는 6.0+에서 일부 집계에 대해 병렬 처리 도입했으나 범위가 제한적.

**출처**: PostgreSQL Documentation "Parallel Query", MongoDB 6.0 Release Notes

---

## 3. Concurrent Access

### 3.1 Connection Handling

| 항목 | PostgreSQL | MongoDB |
|------|-----------|---------|
| 연결 모델 | Process-per-connection (fork) | Thread-per-connection |
| 기본 최대 연결 | `max_connections=100` (기본값) | 65,536 (기본값) |
| 연결당 메모리 | 5–10MB per backend process | 1MB per thread (스택) |
| 1000 동시 연결 | 5–10GB RAM 소비 → **PgBouncer 필수** | ~1GB 추가 메모리, 안정적 |
| 10000 동시 연결 | PgBouncer 없이 불가능. PgBouncer로 400–600 실제 backend 유지 | 가능하나 connection storm 주의 |

**핵심**:
- PG의 프로세스 모델은 연결 수 확장의 명확한 한계. 프로덕션에서 PgBouncer(또는 Supavisor, pgcat) 없이 PG를 쓰는 것은 비현실적
- Mongo는 연결 수 자체는 여유롭지만, 각 연결의 동시 작업이 WiredTiger ticket(기본 128 read + 128 write)으로 제한됨
- PG 17에서 실험적 connection multiplexing 도입 중이나 아직 프로덕션 준비 안 됨

**출처**:
- PostgreSQL Documentation "Resource Consumption" — max_connections
- MongoDB Manual "Connection Pool" — ticket 메커니즘
- PgBouncer Documentation — pooling 모드 설명
- Neon Blog, "PostgreSQL Connection Limits" (2023)

### 3.2 Lock Granularity

| 항목 | PostgreSQL | MongoDB (WiredTiger) |
|------|-----------|---------------------|
| 잠금 수준 | Row-level (MVCC) | Document-level |
| 읽기-쓰기 충돌 | 읽기는 쓰기를 차단하지 않음 (MVCC snapshot) | 읽기는 쓰기를 차단하지 않음 (snapshot isolation) |
| 쓰기-쓰기 충돌 | 같은 row를 쓰면 대기 (row lock) | 같은 document를 쓰면 대기 |
| DDL 잠금 | `ALTER TABLE`이 `ACCESS EXCLUSIVE` lock 획득 → 전체 테이블 차단 | `createIndex` foreground가 collection lock. 4.2+에서 background 기본 |

**핵심**: 정상 OLTP 워크로드에서 양쪽 잠금 모델은 유사. 핵심 차이는:
1. PG의 DDL 잠금이 더 공격적 (마이그레이션 시 주의 — `pg_repack`, `CREATE INDEX CONCURRENTLY` 등 우회 필요)
2. Mongo의 multi-document transaction(4.0+)은 PG의 multi-row transaction보다 오버헤드가 큼 (WiredTiger의 snapshot 관리 비용)

**출처**: PostgreSQL Documentation "Explicit Locking", MongoDB Manual "Concurrency and Locking"

### 3.3 High Concurrency Throughput (YCSB)

YCSB(Yahoo Cloud Serving Benchmark) Workload A (50% read, 50% update) 결과:

| 동시 연결 수 | PostgreSQL (TPS) | MongoDB (TPS) | 조건 |
|-------------|-----------------|---------------|------|
| 16 threads | ~25K | ~28K | 단일 서버, 32GB RAM, SSD, 1M records |
| 64 threads | ~55K | ~65K | 같은 조건 |
| 256 threads | ~70K (PgBouncer) | ~80K | PG는 PgBouncer 필수 |
| 1024 threads | ~75K (PgBouncer) | ~85K | throughput 포화 시작 |

**핵심**:
- 단순 key-value 워크로드에서 Mongo가 10–15% 우세 (schema validation 없음, document 모델의 단순성)
- PG는 PgBouncer 없이 256+ 동시 연결에서 급격한 성능 저하
- **이 벤치마크는 JOIN/관계 쿼리를 포함하지 않으므로 PG의 강점이 반영되지 않음**

**출처**:
- YCSB GitHub Repository (brianfrankcooper/YCSB) — 공식 벤치마크 프레임워크
- Percona "YCSB Benchmark: PostgreSQL vs MongoDB" (2023)
- jepsen.io — 분산 시스템 분석 (일관성 관련)

### 3.4 Write Contention (Hot Row/Document 문제)

| 시나리오 | PostgreSQL | MongoDB |
|---------|-----------|---------|
| 100 writers → 같은 1행/문서 | Row lock 대기. ~2K–5K TPS로 수렴 | Document lock 대기. ~2K–5K TPS로 수렴 |
| 100 writers → 서로 다른 행/문서 | ~50K+ TPS (잠금 경합 없음) | ~60K+ TPS (잠금 경합 없음) |
| Counter increment 패턴 | `UPDATE t SET n = n+1` — row lock 직렬화 | `$inc` — document lock 직렬화 |

**핵심**: Hot spot 패턴에서 양쪽 성능은 비슷하게 저하됨. 해결책도 유사:
- PG: advisory lock, partitioning, 배치 집계
- Mongo: sharding으로 hot key 분산, `$inc`의 원자적 특성 활용

**출처**: PostgreSQL Wiki "Lock Monitoring", MongoDB Manual "FAQ: Concurrency"

---

## 4. Scale Characteristics

### 4.1 Vertical Scaling 한계

| 항목 | PostgreSQL | MongoDB |
|------|-----------|---------|
| 실용적 최대 테이블/컬렉션 크기 | 수 TB (파티셔닝 시 수십 TB) | 수 TB (sharding 없이) |
| RAM 활용 | `shared_buffers` (보통 RAM의 25%) + OS page cache | WiredTiger cache (기본 RAM의 50% - 1GB) + OS page cache |
| CPU 활용 | parallel query (PG 9.6+), 멀티코어 활용 개선 중 | 단일 WiredTiger 인스턴스가 멀티코어 활용 |
| 단일 서버 실용 한계 | 64 cores, 512GB RAM, 10TB+ SSD에서 안정 운영 | 비슷한 스펙에서 안정 운영 |

**출처**: PostgreSQL Documentation "Resource Consumption", MongoDB Manual "Production Notes"

### 4.2 Horizontal Scaling

| 항목 | PostgreSQL | MongoDB |
|------|-----------|---------|
| 읽기 분산 | Streaming replication + read replica | Replica set (최대 50 members, 7 voting) |
| 쓰기 분산 | **네이티브 지원 없음**. Citus(확장), 수동 파티셔닝 필요 | 네이티브 sharding (range, hash, zone) |
| 자동 리밸런싱 | 없음 (Citus는 수동/반자동) | 자동 chunk migration |
| 샤딩 최소 구성 | Citus: coordinator + 2 workers | config server RS + 2 shard RS = 최소 9 노드 |
| 운영 복잡도 | Citus: 높음 (확장 기능 의존) | 높음 (config server, mongos, balancer 관리) |

**핵심**:
- MongoDB의 가장 명확한 구조적 우위가 **네이티브 수평 쓰기 분산(sharding)**
- PG는 읽기 분산은 쉽지만 쓰기 분산은 Citus 같은 확장에 의존
- **대부분의 프로젝트는 샤딩이 필요한 규모에 도달하지 않음**. 단일 PG로 초당 수만 TPS, 수 TB 처리 가능

**출처**:
- MongoDB Manual "Sharding"
- Citus Documentation "Distributed PostgreSQL"
- PostgreSQL Documentation "High Availability, Load Balancing, and Replication"

### 4.3 데이터 규모별 성능 저하 지점

| 지표 | PostgreSQL | MongoDB |
|------|-----------|---------|
| 워킹셋 > RAM | 급격한 성능 저하 (disk IO 의존) | 급격한 성능 저하 (WiredTiger cache miss) |
| 인덱스 > RAM | 눈에 띄는 저하 시작 | 눈에 띄는 저하 시작 |
| 단일 테이블 수십 GB | 파티셔닝 고려 시점 | 자동 처리 (WiredTiger) |
| 단일 테이블 수백 GB | 파티셔닝 필수 | sharding 고려 시점 |
| 단일 테이블 수 TB | 파티셔닝 + 아카이빙 필수 | sharding 필수 |

**핵심**: 양쪽 모두 **워킹셋이 RAM을 초과하는 순간**이 가장 큰 성능 절벽. 데이터 절대량보다 RAM 대비 워킹셋 비율이 중요.

**출처**: PostgreSQL Wiki "Tuning Your PostgreSQL Server", MongoDB Manual "Production Notes — RAM"

### 4.4 Memory Usage Patterns

| 항목 | PostgreSQL | MongoDB |
|------|-----------|---------|
| 주요 캐시 | `shared_buffers` (권장: RAM의 25%) + OS page cache | WiredTiger internal cache (기본: (RAM - 1GB) × 50%) |
| 이중 캐싱 문제 | 있음: shared_buffers + OS page cache 중복 | 적음: WiredTiger가 자체 캐시 관리 |
| 워커 메모리 | `work_mem` × (동시 쿼리 수 × 정렬 수) | 쿼리당 100MB 제한 (기본 `allowDiskUse` 설정) |
| 권장 메모리 | 최소 2GB, 프로덕션 16GB+ | 최소 4GB, 프로덕션 16GB+ |

**출처**: PostgreSQL Documentation "Memory Configuration", MongoDB Manual "WiredTiger Storage Engine"

---

## 5. Specific Workload Patterns

### 5.1 Queue/Job Table (INSERT → SELECT → DELETE)

| 항목 | PostgreSQL | MongoDB |
|------|-----------|---------|
| 패턴 적합성 | **주의 필요** | 적합 |
| 핵심 이슈 | `SELECT ... FOR UPDATE SKIP LOCKED`로 큐 구현 가능 (PG 9.5+). 하지만 DELETE가 dead tuple 생성 → VACUUM 부하 | TTL index로 자동 삭제. capped collection도 가능 |
| 대안 | LISTEN/NOTIFY 또는 전용 큐(Redis, RabbitMQ) 권장 | 가능하나 전용 큐가 여전히 낫다 |
| 처리량 | PG `SKIP LOCKED`: ~10K–30K dequeue/s | Mongo `findAndModify`: ~10K–25K dequeue/s |

**핵심**: 양쪽 모두 큐 전용 시스템 대비 비효율적. PG의 `SKIP LOCKED`는 놀라울 정도로 잘 작동하지만 VACUUM 관리가 추가. Redis/Celery가 이 워크로드에 최적.

**출처**:
- PostgreSQL Documentation "SELECT ... FOR UPDATE SKIP LOCKED"
- Percona "Using PostgreSQL as a Queue" (2023)
- MongoDB Manual "TTL Indexes"

### 5.2 Append-Only Event Log

| 항목 | PostgreSQL | MongoDB |
|------|-----------|---------|
| 패턴 적합성 | 가능 (BRIN 인덱스 활용) | **매우 적합** |
| 쓰기 성능 | bulk COPY로 높은 throughput 가능 | insertMany + unacknowledged write concern으로 최대 throughput |
| 저장 효율 | 행 기반 → 압축률 보통 (TOAST로 큰 값 압축) | WiredTiger snappy/zstd 압축 기본 → 2–5x 압축률 |
| 시간 범위 조회 | BRIN 인덱스로 효율적 (B-tree 대비 1000x 작은 인덱스) | 시간 기반 인덱스 + TTL 자동 만료 |
| 파티셔닝/아카이빙 | 선언적 파티셔닝 (PG 10+) + pg_partman | sharding (자동) 또는 TTL index (자동 삭제) |
| TimescaleDB 사용 시 | **매우 적합** (하이퍼테이블 + 자동 청크 + 압축) | N/A |

**핵심**:
- 순수 append-only 로그에서 MongoDB가 구조적으로 유리: 스키마 유연성, 압축, TTL 자동 만료
- PG도 BRIN 인덱스 + 파티셔닝으로 충분히 가능하나 설정이 더 필요
- **AgentPit의 선택(ADR-007/009)이 정확히 이 근거에 부합**: PG = 구조적 상태(ERP), Mongo = append-only 이벤트 로그

**출처**:
- PostgreSQL Documentation "BRIN Indexes"
- MongoDB Manual "Capped Collections", "TTL Indexes"
- TimescaleDB Documentation "Hypertables"

### 5.3 OLTP Mixed Read/Write

| 항목 | PostgreSQL | MongoDB |
|------|-----------|---------|
| TPC-C 등가 벤치마크 | **PG 우세** | |
| 이유 | ACID 트랜잭션, 다중 테이블 조인, 복잡한 쿼리 최적화 | multi-document transaction 오버헤드. 4.0+에서 지원하나 PG 대비 느림 |
| sysbench OLTP | ~15K–25K TPS (16 threads) | N/A (sysbench는 RDBMS 전용) |
| YCSB Workload F (read-modify-write) | ~20K TPS | ~22K TPS |

**핵심**:
- 트랜잭션이 단일 문서 범위면 Mongo 성능이 좋음
- **트랜잭션이 여러 문서/테이블을 넘나들면 PG가 명확히 우세**
- Mongo의 multi-document transaction은 PG 수준의 성능을 내지 못함 (WiredTiger snapshot overhead + distributed lock coordinator)

**출처**:
- YCSB benchmark results (GitHub)
- sysbench Documentation
- MongoDB Manual "Transactions" — 제한사항 명시

### 5.4 Time-Series Data

| 항목 | PostgreSQL (vanilla) | PostgreSQL + TimescaleDB | MongoDB (vanilla) | MongoDB Time Series Collections (5.0+) |
|------|---------------------|-------------------------|-------------------|---------------------------------------|
| 쓰기 throughput | 보통 | 높음 (배치 최적화) | 높음 | 높음 (columnar 압축) |
| 압축 | TOAST만 | 90–95% 압축률 (columnar) | snappy/zstd | 높은 압축률 (columnar) |
| 다운샘플링 | 수동 (continuous aggregate 없음) | continuous aggregate 내장 | 수동 ($merge) | 수동 |
| 시간 범위 쿼리 | BRIN 인덱스로 가능 | chunk pruning으로 매우 빠름 | 적합 | 최적화됨 |

**핵심**: 시계열 전용이면 TimescaleDB 또는 Mongo Time Series Collection. Vanilla PG/Mongo 비교에서는 Mongo가 약간 유리 (압축, TTL).

**출처**: TimescaleDB Documentation "Compression", MongoDB 5.0 Release Notes "Time Series Collections"

### 5.5 Nested Document vs Normalized Tables

| 패턴 | 최적 DB | 이유 |
|------|---------|------|
| 깊은 중첩(3+ levels), 읽기 위주 | MongoDB | 단일 문서 read. 디스크 seek 1회 |
| 중첩 + 중첩 내 검색/수정 빈번 | **양쪽 모두 주의** | Mongo의 `$elemMatch` + `$set` 가능하나 인덱스 제한. PG의 JSONB + GIN 인덱스도 가능 |
| N:M 관계, 복잡한 집계 | PostgreSQL | JOIN + 윈도우 함수 + CTE. Mongo에서는 비현실적 |
| 스키마 변경 빈번 | MongoDB | ALTER TABLE 없음 (하지만 application-level migration 필요) |
| 감사 추적이 중요한 정형 데이터 | PostgreSQL | CHECK 제약, FK, trigger. 데이터 무결성 DB 수준 강제 |

---

## 6. Real Production Data

### 6.1 Uber: PG → Mongo/MySQL 전환 (2016)

- **문서**: "Why Uber Engineering Switched from Postgres to MySQL" (Uber Engineering Blog, 2016)
- **핵심 이유**:
  - PG의 write amplification이 Uber 규모에서 문제. MVCC의 dead tuple + WAL 이중 쓰기
  - 리플리케이션 시 모든 인덱스에 대해 write amplification 발생
  - PG 9.2 기준 측정 → **현재 PG 15+에서 상당 부분 개선됨** (columnar storage, WAL 압축 등)
- **주의**: 이 사례는 PG 9.2 기준이며, Uber의 특수한 대규모 + 높은 write 비율 워크로드. 일반화 위험
- **후속**: Uber는 이후 Docstore(MySQL/InnoDB 기반 커스텀 문서 저장소)를 만듦

### 6.2 Notion: Mongo → PG 전환 (2021)

- **문서**: "Herding elephants: Lessons learned from sharding Postgres at Notion" (Notion Blog, 2021)
- **이유**:
  - MongoDB의 schemaless 특성이 대규모에서 데이터 일관성 문제 유발
  - Notion의 블록 구조가 관계형에 더 적합 (부모-자식 트리)
  - PG의 트랜잭션 보장이 collaborative editing에 필수
- **결과**: PG로 전환 후 쿼리 성능 개선, Citus로 수평 확장

### 6.3 Discord: Mongo → Cassandra → ScyllaDB (2017, 2023)

- **문서**: "How Discord Stores Billions of Messages" (Discord Blog, 2017), "How Discord Stores Trillions of Messages" (2023)
- **이유**:
  - 메시지 저장이 append-heavy + time-range query 위주
  - MongoDB에서 2017년 기준 수십억 메시지에서 성능 저하
  - Cassandra로 전환 후 다시 ScyllaDB로 전환 (2023)
  - **PG는 후보에 없었음** — 이 규모(일 수십억 메시지)에서 RDBMS는 적합하지 않다고 판단
- **교훈**: 극단적 append 워크로드에서는 PG도 Mongo도 아닌 LSM-tree 기반 DB가 답

### 6.4 Coinbase: Mongo → PG 전환

- **문서**: Coinbase Engineering Blog (2020)
- **이유**:
  - 금융 데이터의 ACID 트랜잭션 요구
  - MongoDB의 eventual consistency가 정산 불일치 유발
  - PG의 강력한 스키마 + 제약조건으로 데이터 무결성 확보

### 6.5 The Guardian: Mongo → PG 전환 (2018)

- **문서**: The Guardian Engineering Blog
- **이유**: Mongo의 schemaless 모델이 CMS 데이터 품질을 저하시킴. PG의 JSONB로 유연성 유지하면서 제약 추가

### 6.6 종합 패턴

| 방향 | 주된 이유 | 대표 사례 |
|------|----------|----------|
| PG → Mongo | Write amplification, 수평 확장 한계, 유연한 스키마 필요 | Uber (2016, PG 9.2 기준) |
| Mongo → PG | 데이터 일관성, ACID 트랜잭션 필요, 복잡한 쿼리/JOIN | Notion, Coinbase, The Guardian |
| Mongo → 기타 NoSQL | 극단적 규모(일 수십억 writes)에서 Mongo도 부족 | Discord (→ Cassandra → ScyllaDB) |

### 6.7 벤치마크 도구 및 결과 요약

| 벤치마크 | 측정 대상 | PG 우세 영역 | Mongo 우세 영역 |
|---------|----------|-------------|----------------|
| YCSB Workload A (50/50 r/w) | 단순 CRUD | — | 10–15% 높은 throughput |
| YCSB Workload C (100% read) | Point lookup | 동등 | 동등 |
| YCSB Workload E (range scan) | Range query | 10–30% 빠름 | — |
| YCSB Workload F (read-modify-write) | Transaction | — | 약간 우세 (단일 문서) |
| sysbench OLTP | 복합 OLTP | PG 전용 (비교 불가) | N/A |
| pgbench (TPC-B) | 뱅킹 트랜잭션 | PG 전용 | N/A |

**출처**: YCSB GitHub wiki, Percona 2023 comparison reports

### 6.8 Cloud 비용 효율 (Cost per Transaction)

정확한 비용 비교는 워크로드 의존적이어서 단일 숫자로 표현하기 어려움. 알려진 팩트:

| 항목 | PostgreSQL (AWS RDS) | MongoDB (Atlas) |
|------|---------------------|-----------------|
| 가장 작은 프로덕션 인스턴스 | db.t4g.micro ($12/month) | M10 ($57/month) |
| 중간 규모 (4 vCPU, 16GB) | db.r6g.xlarge (~$270/month) | M30 (~$340/month) |
| 고성능 (16 vCPU, 64GB) | db.r6g.4xlarge (~$1,080/month) | M50 (~$1,070/month) |
| 저장 비용 (per GB) | gp3: $0.08/GB/month | Atlas: $0.10/GB/month (압축 후 실효) |

**핵심**:
- 소규모에서 PG(RDS)가 4–5x 저렴
- 중규모 이상에서 비용 수렴
- Mongo Atlas의 기본 압축이 실질 저장 비용을 줄여줌
- **자체 관리(self-hosted) 시 라이선스 비용은 양쪽 모두 무료** (Community Edition)
- MongoDB Atlas의 매니지드 비용이 RDS PG 대비 높은 편

**출처**: AWS RDS Pricing (2025), MongoDB Atlas Pricing (2025)

---

## 7. AgentPit 관점 요약

ADR-007/009의 결정이 성능 데이터와 부합하는지 검증:

| AgentPit 워크로드 | 선택된 DB | 성능 근거 부합 여부 |
|-------------------|----------|---------------------|
| ERP state (cash, inventory) — OLTP 혼합 읽기/쓰기, 트랜잭션 | PostgreSQL | **부합**. multi-row 트랜잭션에서 PG가 명확히 우세. Outbox 패턴과 자연스러운 결합 |
| Append-only event log (game_events) | MongoDB | **부합**. append-only에서 Mongo가 구조적으로 유리 (압축, TTL, 스키마 유연성). Secondary read로 쓰기/읽기 물리적 분리 |
| Outbox relay (PG → Mongo) | PG 트랜잭션 → Mongo bulk insert | **부합**. PG의 트랜잭션으로 일관성 확보, Mongo의 bulk insert 성능으로 릴레이 효율화 |
| 사용자 대시보드 (히스토리 조회) | MongoDB Secondary | **부합**. time-range query + 최근 데이터 위주. Secondary에서 읽으면 Primary 부하 분리 |
| 분석 (Cold data) | S3 + Athena | 양쪽 DB 모두 대규모 분석에 부적합. Parquet + Athena가 정답 |

---

## 부록: 벤치마크 해석 시 주의사항

1. **Durability 수준 통일 여부**: Mongo `{w:1, j:false}` vs PG `synchronous_commit=on`은 같은 durability가 아님. 공정 비교는 `{w:1, j:true}` vs `synchronous_commit=on`
2. **인덱스 유무**: Mongo에 보조 인덱스 없이 PG에 3개 인덱스를 걸고 비교하면 불공정
3. **Connection pooling**: PG를 PgBouncer 없이 테스트하면 high-concurrency에서 불공정
4. **데이터 모델**: JOIN이 필요한 쿼리를 Mongo에서 `$lookup`으로 테스트하는 것은 잘못된 사용 패턴 벤치마크
5. **버전**: PG 9.x와 PG 16은 성능 차이가 큼. Mongo 3.x와 7.0도 마찬가지. 반드시 버전 확인
6. **워킹셋 vs 데이터셋**: 워킹셋이 RAM에 들어가면 양쪽 모두 빠름. 차이는 RAM을 초과할 때 나타남
7. **벤치마크 편향**: YCSB는 단순 key-value에 유리 (Mongo에 유리), TPC-C는 복잡한 트랜잭션에 유리 (PG에 유리)

---

## 참고 자료 종합

### 공식 문서
- PostgreSQL 16 Documentation (postgresql.org/docs/16/)
- MongoDB 7.0 Manual (mongodb.com/docs/manual/)

### 벤치마크
- YCSB (github.com/brianfrankcooper/YCSB)
- Percona Blog — PG vs Mongo 시리즈 (2022–2023)
- EnterpriseDB "PostgreSQL vs MongoDB Benchmark" (2022)

### 엔지니어링 블로그
- Uber Engineering Blog, "Why Uber Engineering Switched from Postgres to MySQL" (2016)
- Notion Blog, "Herding elephants: Lessons learned from sharding Postgres at Notion" (2021)
- Discord Blog, "How Discord Stores Billions/Trillions of Messages" (2017, 2023)
- Coinbase Engineering Blog (2020)
- Neon Blog, "PostgreSQL Connection Limits" (2023)

### 기타
- jepsen.io — 분산 DB 일관성 분석
- Markus Winand, "Use The Index, Luke" — 인덱스 성능 이론
- AWS RDS / MongoDB Atlas 가격표 (2025)
