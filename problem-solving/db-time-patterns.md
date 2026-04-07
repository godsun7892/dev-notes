# DB 시간 기반 데이터 처리 패턴

> 유통기한, 생산 타이머 등 시간 경과에 따라 변하는 데이터를 대규모로 처리하는 방법

## 문제

턴마다 수십만 행을 UPDATE하면 DB가 죽는다.

```sql
-- X: 매 턴 30만 행 UPDATE
UPDATE productions SET remaining = remaining - 1;
UPDATE inventory SET expiry = expiry - 1;
```

## 해결: Deadline-based 패턴 (업계 표준)

"남은 턴"을 저장하지 말고 **"완료/만료 시점"을 저장**한다.

```sql
-- O: INSERT 1회, UPDATE 0회
INSERT INTO productions (agent_id, recipe, start_turn, complete_turn)
VALUES ('agent_A', 'baking', 10, 13);

INSERT INTO inventory (item, created_turn, expires_turn)
VALUES ('milk', 10, 12);

-- 읽을 때만 판단
SELECT * FROM productions WHERE complete_turn <= :current_turn;
SELECT * FROM inventory WHERE expires_turn > :current_turn;  -- 안 썩은 것만
```

### 왜 이게 좋은가
- Write 부하 제거: N개 엔티티 × M턴 UPDATE → INSERT 1회
- 서버 다운되어도 시간이 정확히 흐름
- 엔티티 수가 늘어나도 턴당 비용 증가 없음

### 실제 사용처
| 게임 | 적용 |
|---|---|
| EVE Online | 생산 작업을 start_time + duration으로 저장, 조회 시 완료 확인 |
| WoW | 쿨다운을 cooldown_end timestamp로 저장 |
| Clash of Clans | 건물 업그레이드를 start_time + duration 저장 |
| Redis | 키 만료를 만료 시각 저장 + lazy evaluation으로 처리 |

## 완료/만료된 데이터 정리

### 절대 하지 말 것
```sql
-- 피크 타임에 무제한 DELETE → 테이블 lock, 서비스 장애
DELETE FROM productions WHERE complete_turn <= 500;
```

### 실전 패턴 (우선순위 순)

**1. Lazy Evaluation + 배치 정리 (AgentPit 권장)**
- 턴마다 DELETE 0개 — 읽을 때 WHERE로 걸러냄
- 주기적으로(세션 종료 시, 100턴마다) 백그라운드 배치 정리

**2. Hot/Cold 아카이브**
```
[productions]       → 진행 중인 것만 (hot)
       ↓ 완료 시
[productions_archive] → 완료 이력 (cold)
       ↓ 보존 기간 만료
    삭제 또는 S3 이동
```

**3. Chunk DELETE (한 번에 조금씩)**
```sql
-- 1000건씩 나눠서 반복
DELETE FROM productions WHERE complete_turn <= 100 LIMIT 1000;
```

**4. 시간 파티셔닝 + DROP (로그/이력 데이터)**
```sql
-- DROP은 즉시 완료, DELETE 필요 없음
DROP TABLE trade_history_2025_01;
```

## 참고 자료
- EVE Online: GDC 2015 "Scalable Server Architecture"
- WoW: TrinityCore DB 스키마 (character_spell_cooldown)
- Redis: 공식 문서 EXPIRE — lazy deletion + active sampling
- Discord: "How Discord Stores Billions of Messages" (2017, 2023)
- 도서: "Designing Data-Intensive Applications" (Martin Kleppmann) Ch.3
