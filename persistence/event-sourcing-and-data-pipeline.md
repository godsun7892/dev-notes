# Event Sourcing과 데이터 파이프라인

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 이벤트 로그를 source of truth로 두고, hot/cold 저장소로 나누어 운영하는 패턴은 어떻게 설계하는가? |

---

## 1. Event Sourcing이란

이벤트 로그가 **유일한 source of truth**. 나머지(사용자 히스토리, 잔액, 분석 데이터)는 전부 이벤트에서 파생.

```
이벤트 로그 (단일 저장소, append-only)
  │
  ├─→ 사용자 대시보드: actor_id로 필터
  ├─→ 잔액/Ledger: 이벤트에서 INCREASE/DECREASE 합산
  ├─→ 분석 리포트: 이벤트 전체 집계
  └─→ 리플레이: 이벤트 순서대로 재생하여 상태 복원
```

핵심: 별도 잔액 테이블, 히스토리 테이블을 따로 만들 필요 없음. **중복 데이터 없음.**

### 검증된 사례

- **EVE Online**: 거래 이벤트 → 플레이어 마켓 히스토리 + 경제 분석 (사내 경제학자가 같은 데이터 사용)
- **Riot Games**: 매치 이벤트 → 플레이어 전적 + 내부 분석 + 안티치트 (GDC 발표)
- **EventStoreDB**: Greg Young의 오픈소스 이벤트 소싱 DB. 금융사에서 프로덕션 사용
- **Axon Framework**: Java/Kotlin Event Sourcing + CQRS 프레임워크. 은행/보험사 사용
- **Banking core systems**: 모든 현대 핀테크가 이벤트 로그 + 파생 잔액 패턴

---

## 2. 쓰기/읽기 트래픽 분리

### 문제

단일 이벤트 테이블에 쓰기(액션 발생) + 읽기(사용자 조회 + 분석)가 동시에 몰림.

### 해결: MongoDB 레플리카셋

```
Primary (쓰기 전용)
  │ 자동 복제 (네트워크 전송, 이벤트당 수백 바이트)
  ├→ Secondary 1 (읽기)
  ├→ Secondary 2 (읽기)
  └→ Secondary 3 (읽기)
```

- 쓰기: Primary에만. append-only INSERT는 document-level locking으로 충돌 없음
- 읽기: Secondary에서. `readPreference: secondaryPreferred` 설정
- 고가용성: Primary 죽으면 Secondary가 승격 (어차피 최소 3대 구성)
- 읽기 분산은 레플리카셋의 부수 효과

### PostgreSQL 대안

- Streaming replication + read replica
- TimescaleDB (시계열 최적화 PostgreSQL 확장)

### 스케일링 단계

| 규모 | 방식 |
|------|------|
| 초기 | 단일 인스턴스 + 인덱스 |
| 중기 | 레플리카셋 / read replica로 읽기 분산 |
| 대규모 (수억 건) | 샤딩 (tenant_id 또는 시간 기준 데이터 분할) |

---

## 3. Hot → Cold 데이터 파이프라인

### 데이터 온도

- **Hot**: 최근 N일, 빠른 OLTP 저장소(MongoDB/PostgreSQL)에서 즉시 조회
- **Cold**: N일 이후, S3에 Parquet으로 저장. Athena/Spark/BigQuery로 분석

### 파이프라인

```
Hot Store (MongoDB/PostgreSQL)
  → 스케줄 잡 (Lambda/cron, 매일)
  → 오래된 이벤트 읽기
  → Parquet 변환 (PyArrow)
  → S3에 쓰기 (year/month/day 파티션)
  → Hot Store에서 삭제 (또는 TTL 인덱스)
```

### Parquet이란

- 열(컬럼) 기반 저장 형식. 분석 쿼리에서 필요한 컬럼만 스캔
- JSON 대비 5~10x 작은 용량 (압축 내장)
- Athena/Spark/BigQuery 전부 네이티브 지원
- 출처: [Apache Parquet 공식 문서](https://parquet.apache.org/), AWS Athena 문서

### S3 파티셔닝

```
s3://my-events/
  year=2026/
    month=04/
      day=09/
        events-2026-04-09.parquet
```

날짜 기반 파티션. 분석 쿼리 대부분이 시간 범위 필터이므로 파티션 프루닝(partition pruning)으로 비용 절감.

다른 파티션 키:
- `year/month/day` (시간 기반, 가장 흔함)
- `tenant_id` (멀티 테넌트)
- `event_type` (타입별 분리, 카디널리티 낮을 때만)

너무 많은 파티션 (수만 개+) 은 Athena의 partition discovery를 느리게 만듦. 보통 일/주 단위가 적당.

---

## 4. 사용자 히스토리 조회 전략

### 시간대별 조회 경로

```
최근 7일 → Hot Store에서 직접 조회 (빠름)
7일 이후 → S3 Parquet에서 Athena 쿼리 (약간 느림)
```

### 세션/배치 종료 시 리포트 생성

작업이 세션/배치 단위라면, 세션 종료 시점에 리포트를 생성하여 S3에 저장:

```
세션 종료 → 해당 세션 이벤트 전부 집계 → 리포트 생성 (JSON/PDF) → S3 저장
```

사용자는 리포트를 언제든 다운로드 가능. DB에 오래된 데이터를 들고 있을 필요 없음.

### Materialized View

자주 조회되는 집계는 미리 계산해두면 빠름:

```sql
CREATE MATERIALIZED VIEW monthly_user_summary AS
SELECT user_id,
       date_trunc('month', occurred_at) AS month,
       COUNT(*) AS event_count,
       SUM(amount) AS total
FROM events
GROUP BY user_id, month;
```

PostgreSQL은 `REFRESH MATERIALIZED VIEW`로 갱신. 실시간이 필요 없으면 매일 1회 refresh.

---

## 5. 이벤트 스키마 설계

### 업계 표준 패턴 (Segment, Amplitude, Mixpanel 공통)

```json
{
  "event_id": "uuid-v7 (시간순 정렬)",
  "event_type": "order.placed (dot notation)",
  "timestamp": "ISO 8601",
  "tenant_id": "테넌트 식별자",
  "actor": {"user_id": "user_123", "session_id": "abc"},
  "properties": {"product_id": "p_456", "quantity": 2, "price": 50},
  "context": {"app_version": "0.1.0", "platform": "ios"}
}
```

- `event_type` dot notation: 카테고리별 필터 가능 (`order.*`, `payment.*`)
- `properties` 자유 형식: 이벤트별 다른 필드. MongoDB/JSONB에서 자연스러움
- UUID v7: 시간순 정렬 가능 (RFC 9562, 2024년 표준화). 인덱스 효율 + 충돌 방지

### 기록 대상: 액터가 선택한 행동만

| 기록 O (행동) | 기록 X (파생 가능) |
|--------------|------------------|
| order.placed | payment.calculated (price + tax에서 파생) |
| payment.attempted/succeeded/failed | balance.updated (이벤트 합산으로 계산) |
| user.signup | inventory.changed (이벤트에서 재계산) |
| message.sent | session.ended (events에서 도출 가능) |

원칙: **결정 행동만 저장**. 그 결정으로 인한 결과 상태는 derived view에서 계산.

---

## 6. 이벤트 진화 (Schema Evolution)

이벤트는 append-only이므로 **스키마 변경은 backward-compatible**해야 한다.

### 안전한 변경

- 새 필드 **추가** (기본값 있음): OK
- 기존 필드 이름 변경: ❌ (옛 이벤트가 깨짐)
- 기존 필드 타입 변경: ❌
- 기존 필드 삭제: ❌ (옛 이벤트가 의미 잃음)

### 버전 관리

```json
{
  "event_type": "order.placed",
  "schema_version": 2,
  "properties": {...}
}
```

스키마 v2를 추가하면, 처리 코드가 v1과 v2 모두 핸들 가능하게 작성. v1은 영원히 살아있음 (이미 발생한 이벤트).

---

## 7. 안티패턴

### 7.1 이벤트를 mutable하게 다루기
이벤트는 append-only. UPDATE/DELETE 절대 X. 잘못 기록한 이벤트는 보정 이벤트(`order.cancelled`)로 상쇄.

### 7.2 derived state를 source of truth로 착각
잔액 테이블이 source of truth가 되면 이벤트와 어긋났을 때 어느 것이 맞는지 모름. **이벤트가 항상 진실, 잔액은 derived view**.

### 7.3 너무 큰 이벤트
이벤트 하나에 전체 상태를 다 박지 마라 (수 KB+). 차이만 기록. 큰 payload는 별도 저장소(S3) + 참조.

### 7.4 동기 dispatch
이벤트 발생 → 동기적으로 모든 consumer 호출 → 한 consumer가 느리면 전체 막힘. 비동기 큐로 dispatch.

### 7.5 hot store가 무한 성장
TTL 인덱스나 cold archival을 처음부터 설계. 안 그러면 1년 후 storage cost 폭발.

---

## 8. References

| 자료 | 내용 |
|------|------|
| [EventStoreDB](https://github.com/EventStore/EventStore) | 오픈소스 이벤트 소싱 DB |
| [Axon Framework](https://github.com/AxonFramework/AxonFramework) | Java Event Sourcing + CQRS |
| [Segment Spec](https://segment.com/docs/connections/spec/) | 이벤트 스키마 업계 표준 |
| [Apache Parquet](https://parquet.apache.org/) | 컬럼 기반 저장 형식 |
| [AWS Athena 파티셔닝](https://docs.aws.amazon.com/athena/latest/ug/partitions.html) | S3 파티션 프루닝 |
| [MongoDB 레플리카셋](https://www.mongodb.com/docs/manual/replication/) | 읽기 분산 |
| [MongoDB Change Streams](https://www.mongodb.com/docs/manual/changeStreams/) | CDC 패턴 |
| [Martin Fowler — Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) | 개념 정의 |
| [Greg Young — CQRS Documents](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf) | CQRS + Event Sourcing 정리 |
| [UUID v7 — RFC 9562](https://www.rfc-editor.org/rfc/rfc9562.html) | 시간순 정렬 가능 UUID |
