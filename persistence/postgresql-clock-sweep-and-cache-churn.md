# PostgreSQL clock-sweep 축출 + 버퍼 캐시 churn

> "책상 꽉 차면 누구를 치우나(clock-sweep)" 와 "분석 쿼리가 hot 을 쫓아내는 churn".
> 선행: [shared_buffers & 페이지 조회](postgresql-shared-buffers-and-page-lookup.md). 다음: [ring buffer & 한계](postgresql-ring-buffer-strategy-and-limits.md).

---

## 1. 왜 진짜 LRU 를 안 쓰나

가장 공정한 축출 = LRU(가장 오래 안 쓴 놈). 그런데 진짜 LRU 는 **접근할 때마다 그 페이지를 맨 앞으로 이동** → 전역 순서 리스트에 **매 접근 lock** → 수백 프로세스 경합 지옥.
→ PG 는 **근사 LRU = clock-sweep (second chance)**. **카운터 하나만** 만져서 쌈.

---

## 2. clock-sweep 메커니즘 (원형 책상 + 청소부 바늘)

슬롯들이 원형(시계 문자판), 청소부(바늘) 한 명이 돎.

- 각 슬롯 `usage_count` (0~5), 읽힐 때 +1.
- 빈 슬롯 필요 → 바늘이 한 칸씩 전진:

```
각 슬롯 앞에서:
 ┌ refcount>0 (사용 중)      → 건너뜀 (절대 못 치움)
 ├ usage_count>0            → 한 획 깎고 전진 ("한 번 더 봐줌")
 └ usage_count==0 & 안 씀   → ★축출(victim)★
```

- 자주 읽는 hot 은 바늘 오기 전 다시 +1 → 생존. 한 번 읽고 만 cold 는 0 으로 말라 축출.
- `usage_count` 상한 **5** = 인기 식으면 5바퀴 안에 정리.
- dirty 축출은 **디스크 저장부터** → 더 비쌈.

**hot working set 이 캐시에 사는 이유** = 활성 OLTP row + 인덱스 상위가 계속 +1 → 청소부가 매번 봐줌. clock-sweep 이 hot/cold 를 자동 분리.

---

## 3. churn 이란 무엇인가

> **버퍼 캐시 churn = `shared_buffers` 내용물이 너무 빨리 축출·재적재를 반복해 hot 데이터가 캐시에 머물지 못하는 상태.** = "캐시가 캐시 노릇을 못 함."

(churn = 휘젓다 / 미친 회전율)

---

## 4. churn 메커니즘

```
1. 분석 쿼리가 cold 페이지 수백만 개 요구 → 거의 다 MISS
2. MISS 마다 victim 필요 → 청소부가 16,000칸 원을 수십~수백 바퀴 회전
3. 한 바퀴에 usage_count -1. 여러 바퀴 빠르게 → hot(정자5)도 다시 읽히기 전에
   5번 깎여 0 → 축출
4. cold 분석 페이지가 책상 점령, hot working set 쫓겨남
```

### 대가
- 분석 끝나면 OLTP hot 페이지 증발 → 다음 트랜잭션 MISS → µs→ms. 캐시 재가열(re-warm) 비용.
- hit ratio 급락, OLTP p99 튐.

---

## 5. 관측 (실제 쿼리 + 함정)

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

---

## AgentPit 메모

관전(분석) 여럿이 이벤트 집계 → 이벤트 히스토리 페이지가 책상 점령 → **게임 상태 hot set 축출 → 라이브 게임 트랜잭션 느려짐.** 분석이 살아있는 게임을 오염.
→ 완화: 관전이 작은 요약(MV)만 읽게 / 분석을 OLAP 로 분리 (→ [OLTP vs OLAP](oltp-vs-olap-and-customer-facing-analytics.md)).

## 관련 노트
- [shared_buffers & 페이지 조회](postgresql-shared-buffers-and-page-lookup.md)
- [ring buffer strategy & limits](postgresql-ring-buffer-strategy-and-limits.md) — PG 의 churn 방어책이 왜 부족한가
