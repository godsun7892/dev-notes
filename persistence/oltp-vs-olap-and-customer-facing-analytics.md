# OLTP vs OLAP + 고객 대면 분석 (2-Stage 전략)

> "왜 분석 쿼리가 트랜잭션 DB 를 망가뜨리나, 언제 OLAP 로 분리하나" 의 전략 노트.
> 출처: [ClickHouse — Real-time analytics on Postgres](https://clickhouse.com/resources/engineering/real-time-analytics-postgres) 를 읽으며 정리.
> 메커니즘 심화는 자매 노트들(churn / ring buffer / row-store) 참조.

---

## 1. 문제: OLTP 와 OLAP 가 한 DB 에서 싸운다

- SaaS 는 **Postgres 를 source of truth**(회원/거래/상태) 로 씀 = OLTP(짧은 read/write) 최적.
- 그 데이터를 **제품 안 실시간 대시보드**로 사용자에게 되돌려 보여주면 → **분석(OLAP) 쿼리가 같은 PG 에서 트랜잭션과 자원(버퍼 캐시·CPU·커넥션) 경합** → 트랜잭션이 느려지거나 멈춤("grind to a halt").
- 하나의 DB 로 OLTP+OLAP 를 동시에 잘하기 어렵다 — 이게 출발점.

---

## 2. 고객 대면 분석 ≠ 내부 BI (네 워크로드 구분)

> 대부분의 DB 성능 escalation 은 **모든 조회를 하나의 "분석" 으로 뭉뚱그려서** 발생.

| 워크로드 | 성격 |
|---------|------|
| OLTP | 핵심 앱 구동, row 단위 read/write |
| 내부 BI | 임원/분석가 집계, **몇 분 걸려도 안전** |
| ad-hoc 데이터 사이언스 | 노트북 탐색 |
| **고객 대면 분석** | 최종 사용자에게 **직접 노출되는 인-프로덕트 대시보드** |

**고객 대면이 가장 까다로움** — hot OLTP 같은 **sub-100ms** + BI 같은 **복잡 집계** + **테넌트 격리** + **활성 유저 수에 선형 비례하는 동시성.**

### 8 non-negotiables (요약)
sub-100ms p95 / 높은 동시성 / SLA 등급 freshness / 엄격한 테넌트 격리 / 예측 가능한 비용 (… 이걸 "PG 최적화 vs OLAP 이동" 판단 벤치마크로).

---

## 3. 2-Stage 접근

| 단계 | 구성 |
|------|------|
| **Stage 1** | Postgres 단독 + materialized view + 복합 인덱스(tenant_id 우선) + RLS + BRIN |
| **트리거** | breaking point 2개+ 충족 시 졸업 |
| **Stage 2** | PG(source of truth) + **CDC → 컬럼형 OLAP(ClickHouse)**, 대시보드는 OLAP 에서 서빙 |

> **마이그레이션은 재플랫폼이 아니라 PG 옆에 OLAP 를 켜는 additive 작업.** ClickHouse 본인들: "You don't migrate from Postgres. You activate ClickHouse next to it … a configuration change rather than a quarter of pipeline engineering." SQL/스키마 portable.

---

## 4. breaking point — 2개+ 터지면 졸업

| 신호 | 의미 | 심화 |
|------|------|------|
| **버퍼 캐시 churn** | 분석이 hot OLTP 페이지 축출 → 트랜잭션 오염 | [churn 노트](postgresql-clock-sweep-and-cache-churn.md) |
| **요청당 백만 row 스캔** | row-store 라 100만 row 읽어 100ms 불가 | [row-store 노트](row-store-vs-columnar-storage.md) |
| **고-cardinality 인덱스 bloat** | distinct 많은 컬럼 B-tree 비대 → 유지비·메모리·성능 저하 | [high-cardinality 노트](postgresql-high-cardinality-and-index-bloat.md) |
| **빡빡한 freshness SLA** | 초 단위 최신 요구인데 MV 갱신은 분 단위 | [freshness 노트](freshness-tradeoff-and-streaming-olap.md) |
| **ad-hoc 필터링** | 임의 차원 슬라이스 → 미리 집계가 못 덮음 → raw 스캔 | — |

→ **현재 AgentPit = 0개** (유저 0, evaluator 미구현). PG 로 충분.

---

## 5. Stage 1 도구함 (PG 를 한계까지)

- **Materialized view / pre-aggregated table**: 매 요청 raw 스캔 대신 **미리 집계한 요약** 읽음(100만→1). churn 1차 방어. `REFRESH` 는 기본 전체 재계산이라 초 단위 갱신엔 부적합 → freshness 트리거와 연결.
- **복합 인덱스 `(tenant_id, ...)`**: 고객 대면은 항상 `WHERE tenant_id=X` → **좌측 접두 규칙**상 tenant_id 가 맨 앞이어야 즉시 한 테넌트로 좁힘. 우리: `(session_id, occurred_at)`.
- **RLS(Row-Level Security)**: 엔진 레벨에서 "정책 맞는 row 만" 강제 → 코드가 WHERE 깜빡해도 유출 차단. 단 정책이 인덱스와 정렬돼야 성능 안 죽음("careful").
- **BRIN**: 블록 범위마다 min/max 요약만 → B-tree 의 수백~수천분의 1 크기. **데이터가 그 컬럼 순 물리 정렬일 때만** 유효. 우리 events = append-only 시간순 → `occurred_at` BRIN 최적.

### 테넌트 격리 = 논리적 분리 (하드웨어 아님)
공유 테이블 + 모든 row 에 `tenant_id` + 쿼리/RLS 로 거름. 격리 스펙트럼(물리 < DB < schema < **row(논리)**) 중 맨 아래. 우리 tenant = session_id/user_id. 관전 화면에 남 세션 이벤트 새면 안 됨.

---

## 6. AgentPit 의사결정 (객관 근거 기준 — 내부 ADR 아님)

- **관전(spectating) = 정확히 customer-facing OLAP.** 관전자 1명 = 동시 분석 쿼리 1개, sub-100ms 인지 예산, 테넌트(세션) 격리 필수.
- **MVP/1AZ**: events **PG 직행** + Stage 1 도구. breaking point 0개라 객관적으로 충분. 중간 계층(큐/어댑터) 0.
- **확장/3AZ**: breaking point 2개+ 시 PG **옆에** ClickHouse(additive) + [CDC](postgresql-cdc-and-stage2-olap.md). 수집은 자체 pod 가 아니라 **Kafka Connect/Vector**(코드 0, 독립 스케일) 우선.
- **"마이그 무서워 처음부터 self-host CH"** = 객관적 안티패턴: 마이그는 싸고(additive/portable), 소규모 self-host 는 TCO·시간 최대(엔지니어 10~20% + 소규모 AWS self-host 가 managed 보다 비쌈). build-once 가 정 필요하면 **self-host 가 아니라 managed CH Cloud.**

---

## 참고
- [ClickHouse — Real-time analytics on Postgres](https://clickhouse.com/resources/engineering/real-time-analytics-postgres)
- [Tinybird — self-host ClickHouse cost 2026](https://www.tinybird.co/blog/self-hosted-clickhouse-cost)

## 관련 노트
- [churn](postgresql-clock-sweep-and-cache-churn.md) · [ring buffer](postgresql-ring-buffer-strategy-and-limits.md) · [row-store vs columnar](row-store-vs-columnar-storage.md) · [CDC & Stage 2](postgresql-cdc-and-stage2-olap.md)
