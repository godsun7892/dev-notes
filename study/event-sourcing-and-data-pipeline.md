# Event Sourcing과 데이터 파이프라인

## 1. Event Sourcing이란

이벤트 로그가 **유일한 source of truth**. 나머지(사용자 히스토리, Ledger, 분석 데이터)는 전부 이벤트에서 파생.

```
이벤트 로그 (단일 저장소, append-only)
  │
  ├─→ 사용자 대시보드: agent_id로 필터
  ├─→ Ledger/잔액: 이벤트에서 BUY/SELL 합산
  ├─→ 평가 분석: 이벤트 전체 집계
  └─→ 리플레이: 이벤트 순서대로 재생하여 상태 복원
```

핵심: 별도 Ledger 테이블, 히스토리 테이블을 따로 만들 필요 없음. 중복 데이터 없음.

### 검증된 사례

- **EVE Online**: 거래 이벤트 → 플레이어 마켓 히스토리 + 경제 분석 (사내 경제학자가 같은 데이터 사용)
- **Riot Games**: 매치 이벤트 → 플레이어 전적 + 내부 분석 + 안티치트 (GDC 발표)
- **EventStoreDB**: Greg Young의 오픈소스 이벤트 소싱 DB. 금융사에서 프로덕션 사용
- **Axon Framework**: Java/Kotlin Event Sourcing + CQRS 프레임워크. 은행/보험사 사용

---

## 2. 쓰기/읽기 트래픽 분리

### 문제

단일 이벤트 테이블에 쓰기(에이전트 행동) + 읽기(사용자 조회 + 분석)가 동시에 몰림.

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

### 스케일링 단계

| 규모 | 방식 |
|------|------|
| 초기 | 레플리카셋 3대 + 인덱스 |
| 중기 | Secondary 추가로 읽기 분산 |
| 대규모 (수억 건) | 샤딩 (session_id 기준 데이터 분할) |

---

## 3. Hot → Cold 데이터 파이프라인

### 데이터 온도

- **Hot**: 최근 N일, MongoDB에서 즉시 조회 가능
- **Cold**: N일 이후, S3에 Parquet으로 저장

### 파이프라인

```
MongoDB (hot)
  → 스케줄 잡 (Lambda/cron, 매일)
  → 오래된 이벤트 읽기
  → Parquet 변환 (PyArrow)
  → S3에 쓰기 (year/month/day 파티션)
  → MongoDB에서 삭제
```

### Parquet이란

- 열(컬럼) 기반 저장 형식. 분석 쿼리에서 필요한 컬럼만 스캔
- JSON 대비 5~10x 작은 용량 (압축 내장)
- Athena/Spark/BigQuery 전부 네이티브 지원
- 출처: Apache Parquet 공식 문서, AWS Athena 문서

### S3 파티셔닝

```
s3://agentpit-events/
  year=2026/
    month=04/
      day=09/
        events-2026-04-09.parquet
```

날짜 기반 파티션. 분석 쿼리 대부분이 시간 범위 필터이므로 파티션 프루닝으로 비용 절감.

---

## 4. 사용자 히스토리 조회 전략

### 시간대별 조회 경로

```
최근 7일 → MongoDB Secondary에서 직접 조회 (빠름)
7일 이후 → S3 Parquet에서 Athena 쿼리 (약간 느림)
```

### 세션 종료 시 리포트 생성

게임이 세션 단위라면, 세션 종료 시점에 리포트를 생성하여 S3에 저장:

```
게임 종료 → 해당 세션 이벤트 전부 집계 → 리포트 생성 (JSON/PDF) → S3 저장
```

사용자는 리포트를 언제든 다운로드 가능. DB에 오래된 데이터를 들고 있을 필요 없음.

---

## 5. 이벤트 스키마 설계

### 업계 표준 패턴 (Segment, Amplitude, Mixpanel 공통)

```json
{
  "event_id": "uuid-v7 (시간순 정렬)",
  "event_type": "trade.executed (dot notation)",
  "timestamp": "ISO 8601",
  "session_id": "세션 식별자",
  "round_number": 5,
  "actor": {"agent_id": "mill_1", "erp_id": "abc"},
  "properties": {"resource": "flour", "quantity": 2, "price": 50},
  "context": {"game_version": "0.1.0"}
}
```

- `event_type` dot notation: 카테고리별 필터 가능 (`trade.*`, `agent.*`)
- `properties` 자유 형식: 이벤트별 다른 필드. MongoDB에서 자연스러움
- UUID v7: 시간순 정렬 가능 (RFC 9562, 2024년 표준화)

### 기록 대상: 에이전트가 선택한 행동만

| 기록 O (행동) | 기록 X (파생 가능) |
|--------------|------------------|
| trade.executed | production.completed (started에서 파생) |
| offer.sent/accepted/rejected | resource.produced (wake + 역할에서 파생) |
| message.sent | resource.expired (expires_at에서 파생) |
| production.started | 잔액 변동 (이벤트 합산으로 계산) |
| npc.sale | inventory 변동 (이벤트에서 재계산) |
| agent.wake | |

---

## 6. 참고 자료

| 자료 | 내용 |
|------|------|
| EventStoreDB | 오픈소스 이벤트 소싱 DB. GitHub: EventStore/EventStore |
| Axon Framework | Java Event Sourcing + CQRS. GitHub: AxonFramework/AxonFramework |
| Segment Spec | 이벤트 스키마 업계 표준. segment.com/docs/connections/spec/ |
| Apache Parquet | 컬럼 기반 저장 형식. parquet.apache.org |
| AWS Athena 파티셔닝 | S3 파티션 프루닝. docs.aws.amazon.com/athena/ |
| MongoDB 레플리카셋 | 읽기 분산. mongodb.com/docs/manual/replication/ |
| MongoDB Change Streams | CDC 패턴. mongodb.com/docs/manual/changeStreams/ |
