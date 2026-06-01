# PostgreSQL 버퍼 캐시 내부 + 고객 대면 분석 (OLTP/OLAP 분리)

> 출발점: "events 를 PG 직행 vs ClickHouse 부터" 를 고민하다가, ClickHouse 의
> [real-time analytics on Postgres](https://clickhouse.com/resources/engineering/real-time-analytics-postgres)
> 글을 파고들며 **"왜 분석 쿼리가 트랜잭션 DB 를 망가뜨리나"** 를 PG 내부 메커니즘까지 학습한 노트.
> AgentPit 의 **실시간 관전(spectating)** 이 정확히 이 글의 *customer-facing analytics* 라서 직격.

---

## 0. 큰 그림 — 왜 이걸 공부했나

- SaaS 는 **Postgres 를 source of truth**(회원/거래/상태) 로 씀 = OLTP 최적.
- 그 데이터를 **제품 안 실시간 대시보드**로 사용자에게 되돌려 보여주면 → **분석(OLAP) 쿼리가 같은 PG 에서 트랜잭션과 자원 경합** → 트랜잭션이 느려지거나 멈춤.
- **고객 대면 분석 ≠ 내부 BI**:
  - 내부 BI = 분석가 몇 명, 몇 분 걸려도 OK, 동시성 낮음.
  - 고객 대면 = **활성 유저 1명 = 동시 쿼리 1개**, **sub-100ms** 예산(웹 인지 한계), 테넌트 격리 필수.
- **AgentPit 직역**: 유저가 자기 에이전트를 실시간 관전 = 고객 대면 OLAP. 관전 부하가 게임 상태(OLTP) PG 와 경합하면 **라이브 게임 트랜잭션이 느려짐**.

### 2-Stage 접근 (ClickHouse 권고, 벤더 중립적으로 타당)

| 단계 | 구성 | 우리 대응 |
|------|------|----------|
| **Stage 1** | Postgres 단독 + materialized view + 복합 인덱스(tenant_id 우선) + RLS + BRIN | MVP: events PG 직행, 관전은 요약/인덱스로 sub-100ms |
| **트리거** | breaking point 2개+ 충족 시 졸업 | 아래 §6 |
| **Stage 2** | PG(source of truth) + CDC → 컬럼형 OLAP(ClickHouse) | 확장: PG **옆에** CH 추가 (additive, "config change") |

> 마이그레이션은 **재플랫폼이 아니라 PG 옆에 OLAP 를 켜는 additive 작업**. ClickHouse 본인들도
> "You don't migrate from Postgres. You activate ClickHouse next to it" 라고 표현. SQL/스키마 portable.
> → "나중 마이그 무서워서 처음부터 self-host CH" 는 객관적으로 손해 (소규모 self-host = TCO 최대 + 솔로 개발자 시간 최대 소모).

---

## 1. shared_buffers — PG 의 내부 메모리 캐시

- PG 는 자기만의 **공유 메모리 페이지 캐시 = `shared_buffers`**. **8KB 페이지(block)** 슬롯 배열. 기본 128MB, 보통 **RAM 25%**.
- 모든 것(테이블=heap, **인덱스**)이 디스크에 8KB 페이지로 저장. **데이터를 만지려면 그 페이지를 먼저 shared_buffers 슬롯으로 올려야** 함.
- **인덱스도 8KB 페이지** — 테이블과 같은 공용 캐시(shared_buffers)에 올라와 같은 규칙으로 경쟁. (index 가 별도 특별한 게 아님)

### 캐시는 사실 2층 (double buffering)

PG 는 기본 buffered I/O → 캐시가 두 겹:

| 층 | 무엇 | 속도 |
|----|------|------|
| L1 `shared_buffers` | PG 자기 캐시 | 가장 빠름 (포인터 반환) |
| L2 OS page cache | 커널이 든 같은 8KB | shared_buffers miss 여도 커널 hit 면 memcpy (RAM 속도) |
| L3 디스크 | 둘 다 miss | SSD ms, HDD 더 |

→ **`shared_buffers` miss ≠ 디스크.** OS 캐시가 받쳐줄 때 많음.

### BufferDesc — 각 슬롯의 포스트잇(메타데이터)

| 항목 | 의미 |
|------|------|
| **tag** | 이 슬롯이 **어느 페이지**인지 = `(relfilenode, fork, block번호)` |
| **usage_count** | 인기도 (0~5). 접근 시 +1 |
| **refcount** | 지금 누가 쓰는 중(pin). >0 이면 축출 금지 |
| **dirty** | 수정됨. 디스크 저장 전엔 못 버림 |

### tag = 디스크 주소 = 이름표 (한 좌표의 두 얼굴)

- 페이지는 디스크에서 **자리를 안 옮김** → "테이블 X 5번 블록" 은 영원히 같은 디스크 위치(집 주소).
- 그래서 그 좌표가 **이름표**(캐시 조회 키) 이면서 **주소**(적재/반납 위치) 를 겸함.
- tag 의 두 역할: ① **조회 키** — "X-5번이 책상에 있나?" ② **I/O 좌표** — miss 적재 / dirty 축출 시 어디서 읽고 쓸지.

---

## 2. 조회 동작 — buffer table 해시 (hit/miss 가 갈리는 지점)

- 책상 슬롯이 ~16,000개(128MB÷8KB). 매 접근마다 전수 검색은 불가.
- **buffer table** = `tag → buffer id` 해시 테이블 (= 책상 입구의 색인 카드함). **데이터가 아니라 "어느 슬롯에 뭐가 있나" 목차**.
- 흐름: tag 해시 → 카드 있나?
  - **HIT**: 슬롯 pin(refcount++) + usage_count++ → 포인터 반환 (µs).
  - **MISS**: victim 슬롯 확보(clock-sweep) → dirty 면 먼저 write → 디스크/OS 에서 적재 → 카드 새로 등록 → pin, usage_count=1.
- 동시성: 공유 해시라 **128 partition lock** 으로 경합 분산.

---

## 3. clock-sweep — 책상 꽉 찼을 때 누구를 치우나

- **진짜 LRU 안 쓰는 이유**: 매 접근마다 순서 리스트 재배치 → 전역 lock → 수백 프로세스 경합 지옥. clock-sweep 은 **카운터 하나만** 만져서 쌈 (근사 LRU = second chance).
- 메커니즘 (원형 책상 + 청소부 바늘 한 명):
  - 슬롯마다 usage_count(0~5), 읽힐 때 +1.
  - 빈 슬롯 필요 → 바늘이 한 칸씩 전진:
    - `refcount>0`(사용 중) → 건너뜀 (못 치움)
    - `usage_count>0` → 한 획 깎고 전진 (한 번 더 봐줌)
    - `usage_count==0 & 안 씀` → **축출(victim)**
  - 자주 읽는 hot 은 바늘 오기 전 다시 +1 → 생존. 한 번 읽고 만 cold 는 0 으로 말라 축출.
- **hot working set 이 캐시에 사는 이유** = 활성 OLTP row + 인덱스 상위 페이지가 계속 +1 → 청소부가 매번 봐줌.
- usage_count 상한 5 = 인기 식으면 5바퀴 안에 정리.
- dirty 축출은 디스크 저장부터 → 더 비쌈.

---

## 4. 버퍼 캐시 churn — 분석 쿼리가 hot 을 쫓아내는 순간

### 정의
> **shared_buffers 내용물이 너무 빨리 축출·재적재를 반복해 hot 데이터가 캐시에 머물지 못하는 상태.** = "캐시가 캐시 노릇을 못 함."

### 메커니즘
1. 분석 쿼리가 **cold 페이지 수백만 개** 요구 → 거의 다 MISS.
2. MISS 마다 victim 필요 → 청소부가 16,000 슬롯 원을 **수십~수백 바퀴** 미친 회전.
3. 한 바퀴에 usage_count -1. 여러 바퀴 빠르게 도니 **hot(정자5)도 다시 읽히기 전에 5번 깎여 0 → 축출.**
4. cold 분석 페이지가 책상 점령, hot working set 쫓겨남.

### 대가
- 분석 끝나면 OLTP hot 페이지 증발 → 다음 트랜잭션 MISS → µs→ms. 캐시 재가열(re-warm) 비용 + 경합.
- hit ratio 급락, OLTP p99 튐.

### AgentPit
관전(분석) 여럿이 이벤트 집계 → 이벤트 히스토리 페이지가 책상 점령 → **게임 상태 hot set 축출 → 라이브 게임 트랜잭션 느려짐.** 분석이 살아있는 게임을 오염.

---

## 5. ring buffer(BAS) — PG 방어책과 4개의 구멍

### 정체
- **Buffer Access Strategy**: "대량 읽지만 재사용 안 할" 데이터를 메인 풀에 안 쏟고 **작은 전용 링(슬롯 몇 개)에 가둬 재활용**. shared_buffers **안의 격리된 구석** (별도 공간 아님).
- 종류: **BAS_BULKREAD 256KB(=32페이지)** — 큰 seq scan(테이블 > shared_buffers/4) / BULKWRITE 16MB / VACUUM 256KB.
- 메커니즘: 32 슬롯을 원형 재사용 — 페이지 N 읽고 처리 후 N+32 가 덮어씀. 수백만 페이지가 32칸 통과, **나머지 책상은 안 건드림.** per-실행(쿼리 하나마다 자기 링).

### 핵심: ring 여부 = "인덱스냐 본문이냐"가 아니라 **"어떻게 읽느냐(통독 vs 점프)"**

| 작업 | 읽는 것 | ring? |
|------|---------|-------|
| 큰 seq scan(통독) | 본문(heap) 처음→끝 | ✅ ring (구석 32칸) |
| index scan(점프) | ① 인덱스 페이지 ② 본문(heap fetch) | ❌ 둘 다 메인 풀 |

→ **같은 본문 페이지**라도 통독으로 읽으면 ring, 색인 점프로 읽으면 메인 풀. 인덱스 페이지는 **항상** 메인 풀(ring 안 씀).

### 왜 고객 대면 분석엔 무력한가 (4 구멍)

| # | 구멍 | 핵심 |
|---|------|------|
| ① | **인덱스 스캔** | 점프는 ring 안 켬 → 인덱스+본문 전부 메인 풀. 작은 테이블 seq scan(<1/4)도 ring 미발동 |
| ② | **동시성** | ring 은 per-쿼리 + 본문만 가둠. 수천 쿼리 × (인덱스 페이지+B-tree 상위+정렬/해시 상태)가 메인 풀 매몰. 동시성은 churn 증폭기 |
| ③ | **링이 작음(32칸)** | ring 전제 = "한 번 보고 뒤 안 봄". join/sort/집계는 재접근·대량 보유 필요 → 32칸 넘쳐 메인 풀 fallback. (②=양의 문제, ③=구조의 문제. 혼자 돌아도 샘) |
| ④ | **dirty/hint bit** | fresh 데이터를 *읽기만* 해도 hint bit 로 페이지 dirty → 즉시 못 덮어씀 → 메인 풀 누수 (§5-1) |

> 결론: ring 은 **"한 명의 큰 통독"** 전용. 고객 대면(수천 동시 + 인덱스 + join/sort + fresh)은 4구멍으로 다 새서 메인 책상 휩쓺.

### 5-1. hint bit — "읽기인데 쓰기" (④의 핵심)

- MVCC: tuple 에 **xmin**(생성 트랜잭션). 가시성 판단하려면 "xmin commit/abort?" 를 **CLOG**(commit log) 에서 확인 — 매번은 비쌈.
- PG 가 한 번 확인하면 답을 **tuple 에 hint bit 도장** → 다음부턴 CLOG 안 봄. **도장 찍기 = 페이지 수정 = dirty.**
- → **fresh row 를 처음 읽는 SELECT 가 hint bit 로 페이지 dirty 화.** 우리 events(갓 append → 바로 관전 스캔)가 이 누수의 정확한 표적. (옛 데이터는 이미 도장 → 재-dirty 안 됨)
- pin(refcount>0) 페이지도 비슷하게 ring 탈출하나 주범은 hint bit.

---

## 6. churn / breaking point 관측 (실제 쿼리 + 함정)

### pg_stat_database — hit ratio
```sql
SELECT datname, blks_hit, blks_read,
       round(100.0*blks_hit/nullif(blks_hit+blks_read,0),2) AS hit_ratio_pct
FROM pg_stat_database WHERE datname = current_database();
```
- 함정1: **`blks_read` ≠ 물리 디스크** (OS 캐시 hit 가능). 진짜 디스크는 `track_io_timing=on` + `blk_read_time` 또는 `iostat`.
- 함정2: **누적값** → 평생 99% 가 분석 시간대 폭락을 가림. **전/후 delta 샘플링** 필요.

### pg_buffercache — 지금 책상에 뭐가 (가장 직관적)
```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
SELECT c.relname, count(*) AS buffers,
       round(100.0*count(*)/(SELECT setting::int FROM pg_settings WHERE name='shared_buffers'),1) AS pct
FROM pg_buffercache b JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname ORDER BY buffers DESC LIMIT 20;

SELECT usagecount, count(*) FROM pg_buffercache GROUP BY usagecount ORDER BY usagecount;
```
- churn: event 테이블이 책상 점령 + usagecount 1 폭증 / 5 감소. **함정: lock 잡는 무거운 쿼리 → 가끔 스냅샷만.**

### pg_stat_statements — 범인 색출
```sql
SELECT query, calls, shared_blks_hit, shared_blks_read,
       round(100.0*shared_blks_read/nullif(shared_blks_hit+shared_blks_read,0),1) AS miss_pct
FROM pg_stat_statements ORDER BY shared_blks_read DESC LIMIT 10;
```

### churn 확정 = 3개 동시 같은 시각
1. 분석 쿼리 구간 ↔ OLTP p99 튐 겹침
2. 같은 구간 hit ratio delta 하락 + OS read IOPS 상승
3. pg_buffercache 가 hot 테이블 축출 보여줌

### 졸업 트리거(2개+ 충족 시 OLAP 로)
buffer 캐시 churn / 요청당 백만-row 스캔 / 고-cardinality 인덱스 bloat / 빡빡한 freshness SLA / ad-hoc 필터링.
→ **현재 AgentPit = 0개** (유저 0, evaluator 미구현). PG 로 충분.

---

## 7. Stage 1 도구함 (PG 를 한계까지)

- **Materialized view / pre-aggregated table**: 매 요청 raw 스캔 대신 **미리 집계한 요약**을 읽음. churn 1차 방어(작은 요약만 읽어 큰 스캔 자체 제거). `REFRESH` 는 기본 전체 재계산이라 초 단위 갱신엔 부적합 → freshness 트리거와 연결.
- **복합 인덱스 `(tenant_id, ...)`**: 고객 대면 쿼리는 항상 `WHERE tenant_id=X` → **좌측 접두 규칙**상 tenant_id 가 맨 앞이어야 즉시 한 테넌트로 좁힘. 우리: `(session_id, occurred_at)` 등.
- **RLS(Row-Level Security)**: 엔진 레벨에서 "정책 맞는 row 만" 강제 → 코드가 WHERE 깜빡해도 유출 차단. 단 정책이 인덱스와 정렬돼야 성능 안 죽음("careful").
- **BRIN(Block Range Index)**: 블록 범위마다 min/max 요약만 저장 → B-tree 의 수백~수천분의 1 크기. **데이터가 그 컬럼 순 물리 정렬일 때만** 유효. 우리 events = append-only 시간순 → `occurred_at` BRIN 최적.

### 테넌트 격리 = 논리적 분리 (하드웨어 아님)
공유 테이블 + 모든 row 에 `tenant_id` + 쿼리/RLS 로 거름. 우리 tenant = session_id/user_id. 관전 화면에 남 세션 이벤트 새면 안 됨.

---

## 8. Stage 2 — CDC + 컬럼형 OLAP

### CDC(Change Data Capture)
- PG 변경(insert/update/delete)을 **WAL 에서 디코딩**해 다른 시스템(ClickHouse)으로 실시간 복제. PG 원본 안 건드림.
- **물리 복제** = WAL 바이트 그대로 → PG read replica 만. **논리 복제** = 플러그인(pgoutput/wal2json)이 row 단위 논리 이벤트로 디코딩 → 비-PG 가능, 변환 가능. CDC = 논리 복제.
- 부품: logical decoding 플러그인 + **replication slot**(소비자 LSN 추적) + Debezium/PeerDB/ClickPipes.

### "replica/warehouse 말고 OLAP" 인 이유
- read replica ✗ = row-store 복사본, 분석 약점 그대로 + lag.
- 웨어하우스(Snowflake/BigQuery) ✗ = 배치 지향, 지연 초~분, 쿼리당 비용 → sub-100ms 고객 대면 불가.
- 실시간 OLAP(ClickHouse) ✓ = "빠름+동시성+신선함" 틈새.

### DIY CDC 디스크 폭발 (무서운 실패 모드)
- slot 의 `restart_lsn` = 소비자가 아직 필요한 가장 오래된 WAL. PG 는 그보다 새 WAL 을 **못 지움.**
- 소비자 죽거나 밀리면 → WAL 무한 누적 → `pg_wal/` 디스크 풀 → **PG 쓰기 불가 다운.** 원본 DB 전체 정지.
- 방어: `max_slot_wal_keep_size`(PG13+, 초과 시 slot 무효화 — 디스크 살리고 slot 끊는 trade-off) + `pg_replication_slots` 의 lag(`현재 LSN - confirmed_flush_lsn`) 알람 + WAL 디스크 헤드룸.
- "managed CDC 써라" = ClickHouse 세일즈 포지셔닝이나, WAL 폭발 자체는 벤더 무관 실재 사고.

---

## 9. row vs columnar — 왜 OLAP 가 분석에 빠른가

| | Row store (PG) | Column store (ClickHouse) |
|--|----------------|---------------------------|
| 저장 | 한 row 의 모든 컬럼 붙여서 | 컬럼마다 따로 + 압축(LZ4/ZSTD) |
| 1컬럼 집계 | row 전체 다 읽음(낭비) | 그 컬럼만 |
| 실행 | row 하나씩 | 벡터화(배치+SIMD) |
| 인덱스 | row 마다 B-tree | granule(8192행)당 마크 1개 = sparse |
| 압축 | 낮음 | 높음(동종 컬럼) |

→ 분석 집계 10~100x. 그래서 OLAP 를 **PG 옆에** 둠 = PG 버리는 게 아니라 **역할 분리.**

---

## 10. 한 줄 결론 (AgentPit 의사결정)

- **MVP/1AZ**: events PG 직행 + Stage 1 도구(요약 view / `(session_id,occurred_at)` 인덱스 / RLS / BRIN). breaking point 0개라 객관적으로 충분. 중간 계층(어댑터/큐) 0.
- **확장/3AZ**: breaking point 2개+ 시 PG **옆에** ClickHouse(additive) + CDC. 수집은 자체 pod 가 아니라 **Kafka Connect/Vector**(코드 0, 독립 스케일) 우선 검토.
- **"마이그 무서워 처음부터 self-host CH"** 는 객관적 안티패턴 — 마이그는 싸고(additive/portable), 소규모 self-host 는 TCO·시간 최대.

---

## 참고
- [ClickHouse — Real-time analytics on Postgres](https://clickhouse.com/resources/engineering/real-time-analytics-postgres)
- [ClickHouse — Kafka 통합 옵션(Connect/Vector/Table Engine/ClickPipes)](https://clickhouse.com/docs/integrations/kafka)
- [Tinybird — self-host ClickHouse cost 2026](https://www.tinybird.co/blog/self-hosted-clickhouse-cost)
- PostgreSQL 공식: shared_buffers / Buffer Access Strategy(`src/backend/storage/buffer/`) / logical replication / hint bits
- 관련 노트: [postgresql-locking-deep-dive.md](postgresql-locking-deep-dive.md), [db-storage-strategy.md](db-storage-strategy.md), [db-write-scaling-patterns.md](db-write-scaling-patterns.md)
