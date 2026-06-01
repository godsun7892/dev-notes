# PostgreSQL MVCC 와 VACUUM (버전·가시성·dead tuple)

> "읽기/쓰기가 서로 안 막게 하려고 버전을 남기는" MVCC, 그 부산물 dead tuple, 청소부 VACUUM.
> dead tuple·hint bit·visibility map 을 한 우산으로 묶는 토대 노트.
> 참조처: [bloat](postgresql-high-cardinality-and-index-bloat.md) / [ring buffer hint bit](postgresql-ring-buffer-strategy-and-limits.md) / [locking](postgresql-locking-deep-dive.md).

---

## 1. MVCC 란? — 왜 필요한가

**MVCC = Multi-Version Concurrency Control.**

- 문제: 여러 트랜잭션이 같은 데이터를 동시에 읽고 씀.
- 순진한 방법: 쓰는 동안 row 잠금 → 읽는 사람 대기 → 느림.
- **MVCC**: 덮어쓰지 않고 **여러 버전을 남김.** 읽는 사람은 **자기 시작 시점에 유효했던 버전**을, 쓰는 사람은 **새 버전**을 봄.
- → **읽기가 쓰기를 안 막고, 쓰기가 읽기를 안 막음.** (PG 동시성의 핵심)

비유: 문서를 고칠 때 지우지 말고, 옛 줄에 **취소선**(누가 지웠는지 표시) 긋고 **새 줄을 아래 추가.**

---

## 2. 구현: 버전 = tuple, xmin/xmax

- 한 row 는 하나 이상의 **tuple(버전)** 으로 heap 에 저장.
- 숨은 시스템 컬럼: **xmin**(이 버전을 *만든* 트랜잭션), **xmax**(이 버전을 *지운/대체한* 트랜잭션, live 면 0).

```
amount 100 → 150 UPDATE:
  옛 tuple: [amount=100, xmin=10, xmax=25]  ← xmax 찍힘 = txn25가 지움 (dead)
  새 tuple: [amount=150, xmin=25, xmax=0 ]  ← 새 버전 (live)
```
- **UPDATE = 덮어쓰기 아님.** 옛 tuple 에 xmax 찍고 **새 tuple INSERT** → 물리적으로 tuple 2개.
- **DELETE** = xmax 만 찍음 (물리 삭제 X).
- **INSERT** = xmin 찍은 새 tuple 1개.

---

## 3. 가시성(visibility)

- 트랜잭션은 시작 시 **스냅샷**(내 시작 때 commit 돼 있던 것) 보유.
- 각 tuple 의 xmin/xmax 를 스냅샷과 대조:
  - xmin 이 내 스냅샷 전 commit + xmax 비어있음 → **보임**(내겐 live).
  - xmax 가 내 스냅샷 전 commit → **죽은 버전, 건너뜀.**
- → 읽는 사람은 자기 버전을 보고, 동시 쓰기의 새 버전은 아직 안 보임. 서로 안 막음.

---

## 4. dead tuple + VACUUM (청소부)

- UPDATE/DELETE 후 남은 옛 tuple = **dead tuple.** 아무 트랜잭션도 못 보게 되면 쓸모 0인데 **물리적으론 잔존** → 쌓이면 **테이블(heap) bloat.**
- **VACUUM** = 가비지 컬렉터. 아무도 못 보는 dead tuple 의 공간을 **회수**(heap **+ 인덱스 둘 다**).
- autovacuum 이 자동 실행하나 **고-write 면 청소보다 쓰레기가 빨리 쌓임** → bloat.

---

## 5. 부속품: hint bit · visibility map (둘 다 MVCC 가속 장치)

- **hint bit**: "xmin commit/abort?" 를 매번 CLOG(commit log) 확인하면 비쌈 → 한 번 확인하면 답을 **tuple 에 도장**. **도장 = 페이지 수정 = dirty.** → fresh row 첫 SELECT 가 페이지 dirty 화 (※ [ring buffer 누수](postgresql-ring-buffer-strategy-and-limits.md)의 ④).
- **visibility map(VM)**: 페이지마다 1비트 — "이 페이지 모든 tuple 이 누구에게나 visible(all-visible)". VACUUM 이 ON. index-only scan 이 heap 생략하려면 이 비트 필요한데, **fresh 데이터는 OFF** → heap fetch 로 fallback.

---

## 6. 영향 정리

| 현상 | MVCC 연결 |
|------|----------|
| 테이블 bloat | dead tuple 누적 (VACUUM 전) |
| 인덱스 bloat (A 원인) | dead tuple 가리키던 **인덱스 엔트리도 dead** → [bloat 노트](postgresql-high-cardinality-and-index-bloat.md) |
| 페이지가 읽기만 해도 dirty | hint bit |
| index-only scan 이 fresh 에서 느림 | VM 미설정 |

---

## AgentPit 메모
- append-only state/events 라 UPDATE 보다 INSERT 위주지만, `offer`/`production_order` 등 **status 전환(UPDATE)** 이 dead tuple 생성 → autovacuum 모니터링 필요.

## 관련 노트
- [B-tree & heap 구조](postgresql-btree-and-heap-structure.md)
- [고-cardinality & index bloat](postgresql-high-cardinality-and-index-bloat.md)
- [PostgreSQL locking 심층](postgresql-locking-deep-dive.md)
