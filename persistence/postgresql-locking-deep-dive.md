# PostgreSQL Locking 심층 정리

## 1. CRUD 연산별 Lock

### SELECT
- **테이블 lock**: `ACCESS SHARE` (가장 약한 lock. `ACCESS EXCLUSIVE`만 충돌)
- **행 lock**: 없음 (`FOR UPDATE/SHARE` 쓰지 않는 한)
- **MVCC**: SELECT는 쓰기를 안 기다리고, 쓰기도 SELECT를 안 기다림. 트랜잭션 시작 시점의 스냅샷을 읽음

### INSERT
- **테이블 lock**: `ROW EXCLUSIVE`
- **행 lock**: 새 행의 `xmin`에 트랜잭션 ID 기록. 다른 트랜잭션에겐 커밋 전까지 안 보임
- **UNIQUE 제약 있을 때**: 같은 키를 동시에 INSERT하면, 먼저 INSERT한 트랜잭션이 커밋/롤백할 때까지 대기
  - 커밋 → 두 번째 트랜잭션이 duplicate key 에러
  - 롤백 → 두 번째 트랜잭션 성공
- **UNIQUE 없으면**: INSERT끼리 서로 안 기다림

### UPDATE
- **테이블 lock**: `ROW EXCLUSIVE`
- **행 lock**: 대상 행에 `FOR UPDATE` 또는 `FOR NO KEY UPDATE` 획득
  - PK/UNIQUE 컬럼 수정 → `FOR UPDATE` (가장 강함)
  - 일반 컬럼 수정 → `FOR NO KEY UPDATE`
- **내부 동작**: 기존 tuple의 `xmax`에 트랜잭션 ID 기록 + 새 tuple 생성 (DELETE + INSERT)
- 이미 다른 트랜잭션이 같은 행을 UPDATE 중이면 대기

### DELETE
- **테이블 lock**: `ROW EXCLUSIVE`
- **행 lock**: `FOR UPDATE` (가장 강함)
- **내부 동작**: tuple을 물리적으로 삭제하지 않음. `xmax`에 트랜잭션 ID만 기록. 실제 제거는 나중에 VACUUM이 처리

### 핵심 원칙: 모든 lock은 트랜잭션 끝까지 유지. 중간에 풀리지 않음.

---

## 2. 테이블 레벨 Lock 모드 (8단계)

이름에 "ROW"가 들어있어도 전부 테이블 레벨 lock임 (역사적 이름).

| Lock 모드 | 획득하는 SQL | 용도 |
|-----------|-------------|------|
| `ACCESS SHARE` | SELECT | 읽기 전용 |
| `ROW SHARE` | SELECT FOR UPDATE/SHARE | 행 레벨 lock 쿼리 |
| `ROW EXCLUSIVE` | INSERT, UPDATE, DELETE | 데이터 변경 |
| `SHARE UPDATE EXCLUSIVE` | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY | 스키마 보호 |
| `SHARE` | CREATE INDEX (일반) | 데이터 변경 차단 |
| `SHARE ROW EXCLUSIVE` | CREATE TRIGGER, ADD FK (NOT VALID 없이) | 데이터 변경 차단 + 자기 배타 |
| `EXCLUSIVE` | REFRESH MATERIALIZED VIEW CONCURRENTLY | 읽기만 허용 |
| `ACCESS EXCLUSIVE` | DROP TABLE, TRUNCATE, ALTER TABLE 대부분, VACUUM FULL | 전부 차단 (SELECT 포함) |

### 충돌 매트릭스

```
                        AS   RS   RE   SUE   S   SRE   E   AE
ACCESS SHARE (AS)                                          X
ROW SHARE (RS)                                        X    X
ROW EXCLUSIVE (RE)                          X    X    X    X
SHARE UPDATE EXCL (SUE)               X     X    X    X    X
SHARE (S)                        X    X          X    X    X
SHARE ROW EXCL (SRE)             X    X     X    X    X    X
EXCLUSIVE (E)               X   X    X     X    X    X    X
ACCESS EXCLUSIVE (AE)   X   X   X    X     X    X    X    X
```

핵심 관찰:
- SELECT(`ACCESS SHARE`)는 `ACCESS EXCLUSIVE`만 충돌 → SELECT는 거의 안 막힘
- INSERT/UPDATE/DELETE(`ROW EXCLUSIVE`)는 서로 충돌 안 함 → DML 동시 실행 가능. 실제 직렬화는 행 레벨에서 발생
- `SHARE UPDATE EXCLUSIVE`는 자기 자신과 충돌 → 같은 테이블에 VACUUM 2개 동시 실행 불가

---

## 3. 행 레벨 Lock

### 내부 구현

행 lock은 shared memory가 아니라 **tuple header에 직접 저장**됨.
- 무제한 행 lock 가능 (메모리 제한 없음)
- 단, `pg_locks`에 안 보임 (디스크에 저장되니까)

#### Tuple Header 구조

| 필드 | 크기 | 용도 |
|------|------|------|
| `xmin` | 4 bytes | 이 tuple을 생성한 트랜잭션 ID |
| `xmax` | 4 bytes | 이 tuple을 삭제/lock한 트랜잭션 ID. 0 = 없음 |
| `infomask` | 2 bytes | 비트 플래그: committed, aborted, lock_only, lock 종류 등 |
| `ctid` | 6 bytes | 다음 버전 tuple의 물리적 위치 (자기 자신이면 최신) |

행 lock 과정:
1. 트랜잭션이 행을 lock하면 `xmax`에 자기 트랜잭션 ID 기록 + `lock_only` 비트 설정 (삭제가 아니라 lock임을 표시)
2. 다른 트랜잭션이 이 행을 만나면 → `xmax`의 트랜잭션 ID에 share lock을 걸고 대기 → 해당 트랜잭션이 커밋/롤백하면 해제

### 4가지 행 Lock 모드

| 모드 | 획득하는 SQL | 충돌 대상 | 용도 |
|------|-------------|----------|------|
| `FOR KEY SHARE` | FK 제약 자동 체크 | FOR UPDATE만 | 가장 약함. PK 변경/삭제만 차단 |
| `FOR SHARE` | SELECT FOR SHARE | FOR NO KEY UPDATE, FOR UPDATE | 모든 수정 차단 |
| `FOR NO KEY UPDATE` | UPDATE (일반 컬럼) | FOR SHARE, 자기 자신, FOR UPDATE | 수정 + 공유 lock 차단 |
| `FOR UPDATE` | DELETE, UPDATE (키 컬럼), SELECT FOR UPDATE | 전부 | 가장 강함 |

핵심 설계: `FOR KEY SHARE`와 `FOR NO KEY UPDATE`는 **호환됨**. 자식 테이블 INSERT 시 부모 행에 `FOR KEY SHARE`를 걸어도, 부모의 일반 컬럼 UPDATE는 안 막힘.

### NOWAIT과 SKIP LOCKED

- `SELECT FOR UPDATE NOWAIT` — lock 못 잡으면 즉시 에러
- `SELECT FOR UPDATE SKIP LOCKED` — lock된 행은 건너뜀. 큐 처리 패턴에 적합 (outbox relay 등)

### MultiXact

여러 트랜잭션이 같은 행에 호환되는 공유 lock을 걸면, 하나의 `xmax`에 여러 트랜잭션 ID를 담을 수 없음. 이때 **MultiXact ID**를 사용:
- `xmax`에 MultiXact ID 저장 (infomask의 `is_multi` 비트 설정)
- MultiXact ID → (트랜잭션 ID, lock 모드) 쌍 목록으로 매핑
- `$PGDATA/pg_multixact/`에 저장

---

## 4. 인덱스 관련 Lock

### 인덱스 생성 시 테이블 Lock

| 연산 | 테이블 Lock | DML 차단? |
|------|------------|----------|
| `CREATE INDEX` | `SHARE` | **차단** (INSERT/UPDATE/DELETE 불가) |
| `CREATE INDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` | 허용 |
| `REINDEX` | `ACCESS EXCLUSIVE` | **전부 차단** |
| `REINDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` | 허용 |

### B-tree 내부 Page Lock

- B-tree는 개별 index page에 **짧은 share/exclusive lock**을 걸음
- 각 행 fetch/insert 후 **즉시 해제**
- deadlock 불가
- 모든 인덱스 타입 중 동시성 최고

### CREATE INDEX CONCURRENTLY 내부 (3 Phase)

1. 카탈로그에 `INVALID` 상태로 인덱스 등록 (짧은 lock)
2. 테이블 전체 스캔 + 인덱스 빌드 (DML 허용, 변경사항 추적)
3. 2단계 중 변경된 행 재스캔 → 인덱스 valid로 전환

실패하면 `INVALID` 인덱스가 남음 → 수동 DROP 필요. 일반 CREATE INDEX의 약 2배 소요.

---

## 5. 제약 조건과 Lock

### 총정리: 제약 조건별 DML 시 lock 영향

| 제약 조건 | DML 시 lock 영향 | 비고 |
|-----------|-----------------|------|
| **NOT NULL** | 없음 | null-flag 비트 체크일 뿐. lock 무관. ✅검증완료 |
| **DEFAULT** | 거의 없음 | 기본값 평가. `nextval()`은 시퀀스 buffer에 짧은 lightweight lock이 걸리지만 실질적 경합 없음. ✅검증완료 |
| **CHECK** | 없음 | 행 단위 표현식 평가. lock 무관. ✅검증완료 |
| **UNIQUE** | **있음** | 같은 키 동시 INSERT 시 한쪽이 상대 트랜잭션 ID에 lock 걸고 대기. partial unique index도 동일 (WHERE 조건 만족하는 행끼리만). ✅검증완료 |
| **FK (자식 INSERT)** | **있음** | 부모 행에 `FOR KEY SHARE` 획득 (PG 9.3+. 이전엔 FOR SHARE로 더 강했음). 부모의 비키 컬럼 UPDATE와 호환. ✅검증완료 |
| **FK (부모 DELETE)** | **있음** | 자식 테이블 참조 확인. FK 컬럼 인덱스 없으면 sequential scan → 느림. ✅검증완료 |
| **FK (ON DELETE CASCADE)** | **있음** | 자식 행에 DELETE 실행 → `FOR UPDATE` 획득. 전체가 같은 트랜잭션 안에서 실행. ✅검증완료 |
| **EXCLUSION** | **있음** | UNIQUE와 원리는 유사하지만 GiST 인덱스 사용. **UNIQUE보다 deadlock 가능성 높음** (낙관적 충돌 감지 방식). `ON CONFLICT DO UPDATE` 지원 안 함 (DO NOTHING만 가능). ✅검증완료 |

### 총정리: 제약 조건 추가(ALTER TABLE) 시 lock

| 연산 | Lock | DML 차단? |
|------|------|----------|
| ADD NOT NULL | `ACCESS EXCLUSIVE` + 전체 스캔 | **차단** |
| ADD NOT NULL (안전 패턴: CHECK → VALIDATE → SET NOT NULL) | `SHARE UPDATE EXCLUSIVE` (검증 중) | 허용 |
| ADD CHECK | `ACCESS EXCLUSIVE` + 전체 스캔 | **차단** |
| ADD CHECK NOT VALID → VALIDATE | `SHARE UPDATE EXCLUSIVE` (검증 중) | 허용 |
| ADD FK | `SHARE ROW EXCLUSIVE` + 전체 스캔 | **차단** |
| ADD FK NOT VALID → VALIDATE | `SHARE UPDATE EXCLUSIVE` (검증 중) | 허용 |
| ADD UNIQUE | `ACCESS EXCLUSIVE` + 인덱스 빌드 | **차단** |

**패턴**: 대형 테이블에 제약 추가 시 항상 `NOT VALID` → `VALIDATE` 2단계로.

### CRUD 외에 lock에 영향을 주는 요소

| 요소 | 영향 | 검증 결과 |
|------|------|----------|
| **인덱스 수** | 인덱스가 많을수록 INSERT/UPDATE 느림. 주 원인은 page lock이 아니라 **I/O + WAL 오버헤드**. lock manager 경합은 극단적 동시성에서만 유의미 | 초안 부정확 → 수정 |
| **VACUUM** | `SHARE UPDATE EXCLUSIVE` — DML과 충돌 안 하지만 DDL과 충돌 | ✅정확 |
| **트리거** | 원래 행의 lock 시간을 연장하는 게 아님(어차피 트랜잭션 끝까지 유지). **다른 테이블/행에 추가 lock을 거는 것**이 문제. BEFORE/AFTER 모두 같은 트랜잭션 내 실행 | 초안 부정확 → 수정 |
| **서브쿼리/JOIN** | 참조하는 테이블마다 `ACCESS SHARE` 획득 | ✅정확 |
| **파티셔닝 (DML)** | INSERT 시 모든 파티션에 `ROW EXCLUSIVE` 획득하지만, ROW EXCLUSIVE끼리 충돌 안 함 → **실질적 차단 없음**. 오버헤드는 lock manager 부담 | 초안 부정확 → 수정 |
| **파티셔닝 (DDL)** | TRUNCATE/CREATE INDEX 등은 모든 파티션에 `ACCESS EXCLUSIVE` → **실질적 차단** | 추가 |
| **HOT UPDATE** | 인덱스 컬럼을 안 건드리고 + 같은 heap page에 공간 있으면 → **인덱스 삽입 자체를 안 함** (lock 회피가 아니라 작업 자체가 없음). `fillfactor` 낮추면 HOT 확률 증가 | 초안 부정확 → 수정 |
| **Generated Column (STORED)** | 기반 컬럼 UPDATE → 생성 컬럼 재계산. 생성 컬럼이 인덱스에 포함돼 있으면 **HOT UPDATE 불가능**해지는 함정 | 누락 → 추가 |
| **TOAST** | 2KB 초과 값은 별도 TOAST 테이블에 저장. 변경 안 된 TOAST 컬럼은 재작성 안 함. TOAST 테이블에도 자체 page lock 발생하지만 실질적 경합 드묾 | 누락 → 추가 |
| **Deferred Constraint** | UNIQUE/PK/FK/EXCLUSION에 `DEFERRABLE INITIALLY DEFERRED` 가능. 인덱스 삽입(+lock)은 즉시, 위반 체크만 커밋 시점. 쿼리 플래너가 유일성 가정 못 함 → 최적화 저하 | 누락 → 추가 |
| **SERIALIZABLE 격리 수준** | SIRead predicate lock 사용. 일반 lock과 완전히 다름 — **절대 block하지 않고 deadlock도 안 남**. 대신 직렬화 이상 감지 시 트랜잭션 abort. 모든 읽기에 SIRead lock 추가 → 메모리 오버헤드 | 누락 → 추가 |
| **INSERT의 행 lock** | INSERT는 전통적 행 lock(FOR UPDATE 등)을 **안 걸음**. 새 행은 MVCC 가시성으로 보호 (xmin). UNIQUE 체크 시에만 트랜잭션 ID lock으로 대기 | 누락 → 추가 |

---

### UNIQUE + INSERT 경합

1. Tx A가 값 X INSERT → `xmin` = A
2. Tx B가 같은 값 X INSERT → A의 트랜잭션 ID에 lock 걸고 대기
3. A 커밋 → B가 duplicate key 에러
4. A 롤백 → B 성공

**Deadlock 위험**: Tx A가 (1, 2), Tx B가 (2, 1) 순서로 INSERT하면 교착 가능.

`INSERT ... ON CONFLICT DO NOTHING`은 실패한 INSERT의 dead tuple 생성을 방지.

### Foreign Key

**자식 INSERT/UPDATE 시**: 부모 행에 `FOR KEY SHARE` 자동 획득 (가장 약한 행 lock)
- 부모의 일반 컬럼 UPDATE는 안 막힘 (`FOR NO KEY UPDATE`와 호환)

**부모 DELETE/UPDATE 시**: 자식 테이블에서 참조 여부 확인
- **FK 컬럼에 인덱스 있으면**: 인덱스 lookup으로 빠르게 확인
- **FK 컬럼에 인덱스 없으면**: 자식 테이블 sequential scan → 느림 (대형 테이블에서 심각)

**FK 추가 안전 패턴**:
```sql
-- 1단계: NOT VALID로 추가 (짧은 ACCESS EXCLUSIVE, 스캔 없음)
ALTER TABLE child ADD CONSTRAINT fk_parent
  FOREIGN KEY (parent_id) REFERENCES parent(id) NOT VALID;
-- 2단계: 검증 (SHARE UPDATE EXCLUSIVE, DML 허용)
ALTER TABLE child VALIDATE CONSTRAINT fk_parent;
```

### CHECK 제약

- 추가 시 `ACCESS EXCLUSIVE`로 전체 스캔 (기존 행 검증)
- `NOT VALID`로 추가하면 스캔 없이 빠르게 추가 → 이후 `VALIDATE CONSTRAINT`로 검증 (DML 허용)
- **일반 DML 시에는 lock 없음**. 단순 표현식 평가일 뿐

### EXCLUSION 제약

- GiST 인덱스 기반. 겹치는 범위 방지에 사용 (예: 예약 시스템)
- UNIQUE와 유사한 lock 동작 — 충돌하는 미커밋 행이 있으면 대기

---

## 6. DDL Lock

### ALTER TABLE Lock 레벨

대부분 `ACCESS EXCLUSIVE`지만, 소요 시간이 다름.

#### 즉시 완료 (짧은 lock)
- `ADD COLUMN` (기본값 없거나, PG 11+ 비휘발성 기본값)
- `DROP COLUMN` (카탈로그만 수정, 물리 제거는 나중)
- `RENAME COLUMN`

#### 테이블 재작성 (긴 lock)
- `ADD COLUMN` + 휘발성 기본값 (예: `DEFAULT gen_random_uuid()`)
- `ALTER COLUMN TYPE` (대부분)
- `SET LOGGED` / `SET UNLOGGED`

#### 약한 Lock 사용하는 DDL
- `VALIDATE CONSTRAINT` → `SHARE UPDATE EXCLUSIVE` (DML 허용)
- `SET STATISTICS`, `SET (fillfactor=...)` → `SHARE UPDATE EXCLUSIVE`

### NOT NULL 안전 추가 패턴

```sql
-- 1. CHECK로 먼저 추가 (ACCESS EXCLUSIVE 짧게)
ALTER TABLE t ADD CONSTRAINT t_col_nn CHECK (col IS NOT NULL) NOT VALID;
-- 2. 검증 (SHARE UPDATE EXCLUSIVE, DML 허용)
ALTER TABLE t VALIDATE CONSTRAINT t_col_nn;
-- 3. PG 12+: CHECK 있으면 SET NOT NULL이 즉시 완료
ALTER TABLE t ALTER COLUMN col SET NOT NULL;
-- 4. CHECK 제거
ALTER TABLE t DROP CONSTRAINT t_col_nn;
```

---

## 7. Deadlock

### 발생 원리

두 트랜잭션이 서로가 가진 lock을 기다리며 순환 대기.

```
Tx1: UPDATE accounts SET bal = bal + 100 WHERE id = 1;  -- row 1 lock
Tx2: UPDATE accounts SET bal = bal + 100 WHERE id = 2;  -- row 2 lock
Tx1: UPDATE accounts SET bal = bal - 100 WHERE id = 2;  -- Tx2 대기
Tx2: UPDATE accounts SET bal = bal - 100 WHERE id = 1;  -- Tx1 대기 → DEADLOCK
```

### PostgreSQL 감지 방식

- 상시 감시가 아님. lock 획득 실패 시 `deadlock_timeout` (기본 1초) 대기 후 감지기 실행
- wait-for graph에서 순환 탐지
- 발견 시 한 트랜잭션을 강제 중단 (`ERROR: deadlock detected`)

### 흔한 패턴

1. **비일관적 lock 순서**: 같은 행들을 다른 순서로 UPDATE
2. **FK + UNIQUE 상호작용**: 자식 INSERT가 부모 lock → 동시에 부모 DELETE가 자식 scan
3. **다중 행 INSERT + UNIQUE**: 겹치는 키 집합을 다른 순서로 INSERT

### 방지 전략

1. **일관된 순서**: 항상 같은 순서로 행/테이블 접근 (예: ID 오름차순)
2. **짧은 트랜잭션**: 트랜잭션이 짧을수록 deadlock 확률 감소
3. **애플리케이션 재시도**: deadlock 에러(`SQLSTATE 40P01`) 시 재시도 로직
4. **NOWAIT / SKIP LOCKED**: 대기 대신 즉시 실패/건너뜀

---

## 8. Advisory Lock

애플리케이션이 정의하는 lock. PostgreSQL은 인프라만 제공, 의미는 애플리케이션이 결정.

### 두 가지 수명

| | 세션 레벨 | 트랜잭션 레벨 |
|---|---|---|
| 획득 | `pg_advisory_lock(key)` | `pg_advisory_xact_lock(key)` |
| 해제 | `pg_advisory_unlock(key)` 또는 세션 종료 | 트랜잭션 종료 시 자동 |
| 특징 | 커밋/롤백 후에도 유지 | 일반 lock처럼 동작 |

### 사용 사례

1. **분산 작업 처리**: `pg_try_advisory_lock(job_id)` → 한 worker만 획득
2. **싱글턴 프로세스**: cron job 중복 실행 방지
3. **애플리케이션 뮤텍스**: 특정 엔티티의 복잡한 다단계 연산 보호

### 주의: 커넥션 풀링

PgBouncer 트랜잭션 모드에서 세션 레벨 advisory lock은 위험 — 다른 요청이 같은 DB 세션을 재사용할 수 있음. 트랜잭션 레벨만 사용할 것.

---

## 9. 실무 함정

### Lock Queue 문제 (가장 위험)

1. 여러 SELECT가 `ACCESS SHARE` 보유 중
2. `ALTER TABLE`이 `ACCESS EXCLUSIVE` 요청 → 대기열에 들어감
3. **새 SELECT도 대기열 뒤에 줄 섬** (대기 중인 `ACCESS EXCLUSIVE`와 충돌하므로)
4. 결과: 모든 쿼리가 막힘 → 테이블 완전 접근 불가 → **장애**

방지:
```sql
SET lock_timeout = '2s';
ALTER TABLE ...;  -- 2초 안에 lock 못 잡으면 포기
```

### Long-Running 트랜잭션

- 모든 lock이 트랜잭션 끝까지 유지
- idle in transaction 상태의 세션이 DDL을 막고, VACUUM도 방해
- dead tuple 축적 → 테이블 bloat

방지:
- `idle_in_transaction_session_timeout` 설정 (예: 5분)
- `statement_timeout` 설정
- `transaction_timeout` (PG 17+)

### Autovacuum 충돌

- 일반 autovacuum은 DML과 충돌 안 함 (`SHARE UPDATE EXCLUSIVE`)
- **Anti-wraparound vacuum**은 양보 안 함 — 다른 연산을 차단하고 끝까지 실행. 대형 테이블에서 수 시간 소요 가능
- 장기 트랜잭션이 있으면 VACUUM이 dead tuple을 못 지움 (해당 트랜잭션의 스냅샷에서 아직 보일 수 있으므로)

---

## 10. 모니터링

### pg_locks 뷰

| 컬럼 | 의미 |
|------|------|
| `locktype` | relation, tuple, transactionid, advisory 등 |
| `relation` | lock된 테이블 OID |
| `mode` | lock 모드명 |
| `granted` | true = 보유 중, false = 대기 중 |
| `pid` | 프로세스 ID |
| `waitstart` | (PG 14+) 대기 시작 시각 |

**주의**: 행 레벨 lock은 tuple header에 저장되므로 `pg_locks`에 안 보임.

### 핵심 모니터링 쿼리

**대기 중인 쿼리 찾기**:
```sql
SELECT relation::regclass, * FROM pg_locks WHERE NOT granted;
```

**누가 누구를 막는지** (PG 9.6+):
```sql
SELECT
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query,
  pg_blocking_pids(blocked.pid) AS blocking_pids,
  blocker.query AS blocking_query
FROM pg_stat_activity blocked
JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS bp(pid) ON true
JOIN pg_stat_activity blocker ON blocker.pid = bp.pid
WHERE blocked.wait_event_type = 'Lock';
```

### 설정 파라미터

| 파라미터 | 기본값 | 용도 | 프로덕션 권장 |
|----------|--------|------|-------------|
| `lock_timeout` | 0 (비활성) | lock 대기 최대 시간 | DDL 시 필수 설정 |
| `deadlock_timeout` | 1s | deadlock 감지 대기 시간 | |
| `log_lock_waits` | off | lock 대기 로깅 | **켜기** |
| `idle_in_transaction_session_timeout` | 0 | idle 트랜잭션 자동 종료 | **설정 필수** |
| `statement_timeout` | 0 | 문 실행 최대 시간 | |

---

## 출처

- PostgreSQL 17 공식 문서: Explicit Locking
- PostgreSQL 공식 문서: pg_locks View, Locking and Indexes
- CYBERTEC: Row Locks in PostgreSQL, Index Your Foreign Key
- Citus Data: 7 Tips for Dealing with PostgreSQL Locks
- pganalyze: Postgres Lock Monitoring, Avoiding Deadlocks in Migrations
- Xata: Schema Changes and the Postgres Lock Queue
- Postgres Professional: MVCC Row Versions (xmin/xmax internals), Row-Level Locks
- AWS: Hidden Dangers of Duplicate Key Violations, MultiXacts in PostgreSQL
