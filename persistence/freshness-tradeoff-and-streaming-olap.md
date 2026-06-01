# Freshness 삼각 trade-off 와 streaming OLAP

> 졸업 신호 ④ "빡빡한 freshness SLA". **미리 집계(materialized view)의 숨은 대가** = 신선도, 그 삼각형, 그리고 ClickHouse 가 그걸 깨는 원리.
> 원문 매핑: §3 non-negotiable **#3 "Tunable freshness per dashboard"** + §4 When Stage 1 breaks **"tighter freshness SLAs"**.
> 토대: [row-store vs columnar](row-store-vs-columnar-storage.md)(②), [OLTP vs OLAP](oltp-vs-olap-and-customer-facing-analytics.md).

---

## 1. freshness 란?
- **freshness = 대시보드가 보여주는 데이터가 얼마나 최신이냐.** "사건 발생 → 화면 반영" 지연.
- **freshness SLA** = 고객 약속 (예: "최대 5초", "실시간", "1분 이내").

---

## 2. Stage 1 의 숨은 대가: 미리 집계 = 빠른 읽기 ↔ 신선도

②백만 row 스캔을 피하려고 **미리 집계(MV/rollup)** → 대시보드는 요약 1줄(100만→1). **그런데 요약은 마지막 refresh 시점만큼만 신선** → 1분마다 refresh 면 데이터 최대 1분 묵음.
→ **미리 집계는 "읽기 속도"를 "신선도"와 맞바꾼 것.**

---

## 3. 삼각 trade-off — 셋 중 2개만 (★)

시나리오: "각 에이전트 누적 거래액" 패널, events 초당 수천 건.

```
        신선(fresh)
         /       \
   [꼭짓점1]   [꼭짓점3]
   빠른읽기 ──[꼭짓점2]── 싼쓰기
```

| 꼭짓점 | 방법 | 희생 |
|--------|------|------|
| **1. 신선+빠른읽기** | 요약 테이블을 **매 INSERT 마다 즉시 갱신**(`UPDATE agent_totals ...`) | **비싼 쓰기** — 같은 요약 row 에 write 몰림 = hot row lock 경합 + write 증폭 |
| **2. 빠른읽기+싼쓰기** | 요약을 **1분마다 REFRESH**, write 는 append 만 | **묵음(stale)** — 최대 1분 지연 |
| **3. 신선+싼쓰기** | 요약 X, read 때 `SELECT SUM(...) WHERE agent_id=X` raw 집계 | **느린 읽기** — ②백만 row 스캔 |

**뿌리는 하나**: "row-store 라 raw 집계가 느림(②)" → 비용을 *쓰기 때(1)·묵힘(2)·읽기 때(3)* 중 하나로 **반드시** 치러야 함.

---

## 4. ClickHouse 가 삼각형을 *깨는* 법

핵심: 삼각형이 존재하는 **유일한 이유 = raw 집계가 느림**. 그걸 빠르게 만들면 삼각형이 사라짐.

- **컬럼형 + 벡터화** → `SUM(amount) GROUP BY agent_id` 가 **amount 컬럼만(8MB) SIMD 합산** → 100만 fresh row 도 한 자릿수 ms.
- → **미리 집계 자체가 불필요** → 신선도 세금 안 냄.
- + **연속 ingest**(async_insert/스트림) → 데이터 계속 흘러듦 → fresh.

```
꼭짓점3(신선+싼쓰기)를 택하되, 약점 '느린 읽기'를 컬럼형으로 제거:
  write: 연속 append (싸다) ✅
  read:  raw 그때그때 집계 → 컬럼형이라 빠름 ✅ + 미리 집계 없으니 안 묵음 ✅
  → 신선 + 빠른읽기 + 싼쓰기 셋 다
```

> 한 줄: **PG = "raw 집계 느림 → 미리 집계 강제 → trade-off". CH = "raw 집계 빠름 → 미리 집계 불필요 → trade-off 없음".**

---

## 5. 정직한 단서 (과장 방지)

1. **극단 규모엔 CH 도 미리 집계** — 수십억 row + 초고-card GROUP BY 면 CH 도 `AggregatingMergeTree`/MV. 단 **"강제" 임계가 PG 보다 훨씬 높음.**
2. **CH "싼 쓰기" = 배치 append** — 작은 단건 INSERT 싫어함(merge 부담). `async_insert` 로 모아 씀.
3. **CH 는 OLTP/UPDATE 약함** — PG 대체 아니라 **옆에 추가**(state=PG, 분석 read=CH). 역할 분리.

---

## 6. Tunable freshness per dashboard (원문 #3 — 실용 포인트)

모든 패널이 같은 신선도를 요구하지 않음 → **패널마다 다르게 튜닝 → 필요한 곳에만 비용 지불.**

### AgentPit 직역 — 관전 패널마다 freshness 정반대
| 패널 | freshness | 데이터량 | 어떻게 |
|------|-----------|---------|--------|
| "방금 행동" 라이브 피드 | **초** | **적음**(최근 N개) | 시간 인덱스 직접 조회 or **Redis Pub/Sub 푸시** — 소량이라 ②문제 없음, PG 로 충분 |
| "누적 거래/승률" 통계 | 분~시간 OK | 큼 | 미리 집계(rollup), 묵어도 무방 / 초 단위+빠름 요구 시 → CH 졸업 |

→ 라이브는 **소량이라 신선해도 싸고**, 누적 스탯은 **대량이라 신선하면 비쌈.** 그래서 둘을 **분리 설계.**

---

## 한 줄
> freshness = 데이터 최신 정도. 미리 집계는 **읽기 속도를 신선도와 맞바꾼 것**, **{신선·빠른읽기·싼쓰기} 중 2개만** 가능(뿌리=raw 집계 느림). CH 는 컬럼형으로 raw 집계를 빠르게 해 **미리 집계를 없애 삼각형을 깸.** 단 freshness 는 **패널마다** 다르게 튜닝.

## 관련 노트
- [row-store vs columnar](row-store-vs-columnar-storage.md) — 왜 raw 집계가 느린가/빠른가
- [OLTP vs OLAP](oltp-vs-olap-and-customer-facing-analytics.md) — 졸업 신호 전체
- [CDC & Stage 2 OLAP](postgresql-cdc-and-stage2-olap.md) — 연속 ingest 경로
