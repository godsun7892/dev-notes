# CDC + Stage 2 실시간 OLAP (PG → ClickHouse)

> Stage 2: PG 를 source of truth 로 두고 변경분만 OLAP 로 흘리는 CDC, 그리고 DIY CDC 의 디스크 폭발.
> 전략 맥락: [OLTP vs OLAP 고객 대면 분석](oltp-vs-olap-and-customer-facing-analytics.md).

---

## 1. CDC (Change Data Capture) 개념

- PG 변경(insert/update/delete)을 **WAL(Write-Ahead Log)** 에서 읽어 다른 시스템(ClickHouse)으로 실시간 복제. **PG 원본 안 건드림.**
- WAL = PG 의 모든 변경을 디스크 데이터보다 먼저 기록하는 **순차 로그**(내구성·크래시 복구용).

---

## 2. 물리 복제 vs 논리 복제

| 복제 종류 | 동작 | 타깃 |
|----------|------|------|
| **물리(physical)** | WAL **바이트 그대로** 스트리밍 → byte-identical 복사 | **PG read replica 만** |
| **논리(logical)** | 플러그인(`pgoutput`/`wal2json`)이 WAL 을 **row 단위 논리 이벤트**(테이블/op/old·new)로 **디코딩** | **비-PG 가능**(ClickHouse), 변환·필터 가능 |

→ **CDC = logical replication.** "원본 안 건드리고 변경분만 디코딩해 CH 로."

### 부품 3개
- **logical decoding 플러그인**: WAL(물리) → 논리 변경 이벤트 번역.
- **replication slot**: 서버측 커서. 소비자가 어디까지 읽었나(**LSN**)를 추적·기억. 연결 끊겨도 위치 보존.
- **Debezium / PeerDB / ClickPipes**: logical decoding 을 떠서 변경을 흘리는 CDC 플랫폼.

흐름: `PG 변경 → WAL → logical decoding → slot 이 LSN 추적 → 커넥터가 읽어 변환 → ClickHouse`.

---

## 3. "replica/warehouse 말고 OLAP" 인 이유

- **read replica ✗** = 물리 복제라 **row-store 똑같은 복사본.** read 동시성은 분산되나 각 분석 쿼리는 여전히 느림 + lag. "OLTP read 더"엔 좋고 "빠른 분석"엔 무용.
- **웨어하우스(Snowflake/BigQuery) ✗** = 컬럼이지만 **배치 지향, 지연 초~분, 쿼리당 비용 큼.** 수천 동시 sub-100ms 고객 대면엔 부적합.
- **실시간 OLAP(ClickHouse/Druid/Pinot) ✓** = "빠름 + 동시성 + 신선함" 틈새.

---

## 4. DIY CDC 디스크 폭발 — 무서운 실패 모드

### 메커니즘
- slot 의 **`restart_lsn`** = 소비자가 아직 필요한 **가장 오래된 WAL 위치.**
- PG 는 `restart_lsn` 보다 **새 WAL 을 절대 삭제 못 함**(소비자가 다시 달라 할 수 있으니).
- 소비자(DIY CDC)가 **죽거나 밀리면** → `restart_lsn` 정지 → WAL 이 `pg_wal/` 에 **무한 누적** → **디스크 풀** → **PG 쓰기 불가 다운.** 원본 DB 전체 정지.

### 방어
- **`max_slot_wal_keep_size`** (PG13+): slot 이 붙들 WAL 상한. 초과 시 **slot 무효화**(소비자 위치 잃고 재-snapshot) — **디스크 살리되 slot 끊는** trade-off.
- **모니터링**: `pg_replication_slots` 의 `active`/`restart_lsn`/`confirmed_flush_lsn`. lag = `현재 LSN - confirmed_flush_lsn` → 커지면 알람.
- **WAL 디스크 헤드룸** 확보.

### 솔직한 단서
"managed CDC 써라" 는 ClickHouse/벤더의 매니지드 세일즈 포지셔닝이기도 하나, **WAL 폭발 자체는 벤더 무관 실재·유명 사고.** 직접 짜면 위 3방어는 비협상.

---

## AgentPit 메모
- breaking point 2개+ 도달 시 PG 옆에 ClickHouse + CDC. 그 전(현재)엔 불필요.
- 수집 경로(Kafka Connect/Vector vs 자체 pod)는 [OLTP vs OLAP 노트 §6](oltp-vs-olap-and-customer-facing-analytics.md) 참조.

## 관련 노트
- [OLTP vs OLAP 고객 대면 분석](oltp-vs-olap-and-customer-facing-analytics.md)
- [row-store vs columnar](row-store-vs-columnar-storage.md)
- [PostgreSQL locking 심층](postgresql-locking-deep-dive.md)
