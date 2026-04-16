# DB 시간 기반 데이터 처리 패턴

> 유통기한, 작업 타이머, 쿨다운 등 시간 경과에 따라 변하는 데이터를 대규모로 처리하는 방법

---

## 1. 문제

매 step/tick마다 수십만 행을 UPDATE하면 DB가 죽는다.

```sql
-- ❌ 매 tick마다 30만 행 UPDATE
UPDATE jobs  SET remaining = remaining - 1;
UPDATE items SET expiry = expiry - 1;
```

엔티티가 1만 개면 1만 UPDATE/tick, 10만 개면 10만 UPDATE/tick. 선형 증가하다가 결국 DB가 못 따라감.

---

## 2. 해결: Deadline-based 패턴 (업계 표준)

"남은 tick"을 저장하지 말고 **"완료/만료 시점"을 저장**한다.

```sql
-- ✅ INSERT 1회, UPDATE 0회
INSERT INTO jobs (account_id, job_type, start_tick, complete_tick)
VALUES ('acc_A', 'task_x', 10, 13);

INSERT INTO items (kind, created_tick, expires_tick)
VALUES ('item_y', 10, 12);

-- 읽을 때만 판단
SELECT * FROM jobs  WHERE complete_tick <= :current_tick;
SELECT * FROM items WHERE expires_tick > :current_tick;  -- 안 만료된 것만
```

### 왜 이게 좋은가
- **Write 부하 제거**: N개 엔티티 × M tick UPDATE → INSERT 1회
- **서버 다운에도 시간 정확**: 시계가 멈추는 게 아님
- **엔티티 수 증가에도 tick당 비용 일정**: O(1)

### 실제 사용처

| 시스템 | 적용 |
|---|---|
| EVE Online | 생산 작업을 `start_time + duration`으로 저장, 조회 시 완료 확인 |
| WoW | 쿨다운을 `cooldown_end` timestamp로 저장 |
| Clash of Clans | 건물 업그레이드를 `start_time + duration` 저장 |
| Redis | 키 만료를 만료 시각 저장 + lazy evaluation으로 처리 |
| AWS DynamoDB | TTL attribute (만료 시각)로 자동 정리 |

---

## 3. Lazy Evaluation의 핵심

```python
# 만료 처리 — 절대 DELETE 하지 않음
def active_items(items, current_tick):
    return [i for i in items if i.expires_tick > current_tick]
```

만료된 행은 그냥 무시. 물리적 삭제는 별도 배치 작업으로.

이 패턴을 쓰면:
- 만료 시점에 0회 작업
- 시간이 지나면 자동으로 "보이지 않게" 됨
- 정리는 한가한 시간에

---

## 4. 완료/만료된 데이터 정리

### 절대 하지 말 것
```sql
-- ❌ 피크 타임에 무제한 DELETE → 테이블 lock, 서비스 장애
DELETE FROM jobs WHERE complete_tick <= 500;
```

수십만 row DELETE는 트랜잭션 락 + WAL 폭증 + 인덱스 갱신으로 운영 영향이 큼.

### 실전 패턴 (우선순위 순)

**1. Lazy Evaluation + 배치 정리 (권장)**
- tick마다 DELETE 0개 — 읽을 때 WHERE로 걸러냄
- 주기적으로(한가한 시간, 100 tick마다) 백그라운드 배치 정리

**2. Hot/Cold 아카이브**
```
[jobs]       → 진행 중인 것만 (hot)
       ↓ 완료 시
[jobs_archive] → 완료 이력 (cold)
       ↓ 보존 기간 만료
    삭제 또는 S3 이동
```

**3. Chunk DELETE (한 번에 조금씩)**
```sql
-- 1000건씩 나눠서 반복
DELETE FROM jobs WHERE complete_tick <= 100 LIMIT 1000;
```
PostgreSQL 13+ `DELETE ... LIMIT`은 직접 지원 안 함, CTE 사용:
```sql
WITH to_delete AS (
    SELECT id FROM jobs WHERE complete_tick <= 100 LIMIT 1000
)
DELETE FROM jobs WHERE id IN (SELECT id FROM to_delete);
```

**4. 시간 파티셔닝 + DROP (로그/이력 데이터)**
```sql
-- 월별 파티션
CREATE TABLE trade_history_2026_01 PARTITION OF trade_history
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- DROP은 즉시 완료, DELETE 필요 없음
DROP TABLE trade_history_2025_01;
```

PostgreSQL의 declarative partitioning, MySQL의 partition pruning, BigQuery의 partitioned tables 모두 같은 패턴.

---

## 5. 인덱스 설계

deadline 기반 패턴은 deadline 컬럼에 인덱스가 필수:

```sql
CREATE INDEX idx_jobs_complete_tick ON jobs (complete_tick);
CREATE INDEX idx_items_expires_tick ON items (expires_tick);
```

쿼리:
```sql
SELECT * FROM jobs WHERE complete_tick <= :now;
-- 인덱스 range scan으로 즉시
```

복합 인덱스 (account_id + deadline)도 좋음:
```sql
CREATE INDEX idx_jobs_account_deadline ON jobs (account_id, complete_tick);
```
"이 계정의 진행 중 작업" 같은 쿼리에 효율적.

---

## 6. 안티패턴

### 6.1 Polling으로 deadline 확인
```python
while True:
    overdue = db.fetch("SELECT * FROM jobs WHERE complete_tick <= NOW()")
    for p in overdue:
        finish_job(p)
    sleep(1)
```
이건 OK. 단, deadline 컬럼에 인덱스 필수. 인덱스 없으면 매번 풀스캔.

### 6.2 Deadline을 저장 안 하고 매번 계산
```python
# ❌ 클라이언트가 매번 계산
job.complete_at = job.start + JOB_DURATIONS[job.type]
```
저장해야 인덱스 쿼리 가능. 계산은 INSERT 시점에 1회만.

### 6.3 Tick 단위가 너무 작음
1초 단위 tick으로 1년치 = 31,536,000 행. 너무 많음. **자연스러운 시간 단위** (5분, 1시간 등) 선택.

---

## 7. References

- EVE Online: GDC 2015 "Scalable Server Architecture"
- WoW: TrinityCore DB 스키마 (`character_spell_cooldown`)
- [Redis EXPIRE — lazy deletion + active sampling](https://redis.io/commands/expire/)
- [DynamoDB TTL](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)
- [PostgreSQL Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
- "How Discord Stores Billions of Messages" (2017, 2023)
- Martin Kleppmann — *Designing Data-Intensive Applications*, Ch.3
