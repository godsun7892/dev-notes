# DB 저장소 분리 전략: RDBMS vs NoSQL

## 판단 기준

| 기준 | RDBMS | NoSQL |
|------|-------|-------|
| 관계 | FK/join 필요 | 독립적, 다른 테이블 참조 불필요 |
| 연산 | 읽기 + 쓰기 + 수정 혼합 | 쓰기 위주 or 읽기 위주 (한쪽) |
| 일관성 | 트랜잭션으로 여러 테이블 묶어야 함 | eventual consistency OK |
| 수정 패턴 | UPDATE/DELETE 빈번 | append-only, 수정 안 함 |

## 핵심 원칙

- **join 성이 짙은 것 + 트랜잭션으로 묶어야 하는 것** → RDBMS
- **join 필요 없고 + 오직 조회나 쓰기만 하는 것** → NoSQL

## 예시 적용 (경제 시뮬레이션)

### RDBMS (PostgreSQL)
- ERP, Agent, Session — 관계 중심, join 필수
- ProductionOrder — ERP에 FK, 상태 변경(QUEUED→DONE)
- ResourceBatch — ERP에 FK, 소비 시 수량 차감

### NoSQL (MongoDB/DynamoDB)
- Transaction (Ledger) — append-only, 수정 없음, agent/session 단위 필터링
- 라운드 이벤트 로그 — 평가 분석용, 쌓기만 함
- 평가 리포트 — 생성 후 읽기만

## 턴제 시스템에서의 이점

턴제(라운드 기반)에서는 실시간 분산 트랜잭션 걱정이 없다.
- 라운드 처리: 인메모리에서 일관성 보장
- 라운드 끝: RDBMS에 최종 상태 flush
- 라운드 끝: NoSQL에 이벤트 append
- NoSQL 쓰기 실패해도 RDBMS가 source of truth이므로 재생성 가능
