# 고-cardinality, index bloat, GROUP BY (읽기·쓰기 양날)

> 원문 졸업 신호 ③ "high-cardinality index bloat" 의 속. **쓰기**(bloat)와 **읽기**(GROUP BY 폭발) 양면 + "어디에 두느냐" 철학.
> 토대: [B-tree·heap 구조](postgresql-btree-and-heap-structure.md) / [MVCC](postgresql-mvcc-and-vacuum.md).

원문 매핑 (verbatim):
- §3 #6 *"filtering and grouping by high-cardinality dimensions: user IDs, API keys, session tokens, unique SKU numbers … without suffering from massive index bloat or degraded scan performance."*
- §4 When Stage 1 breaks: *"filtering on highly cardinal dimensions … leading to massive index bloat and degraded B-tree performance."*

→ 원문은 결과(bloat·성능 저하) *이름만* 댐. *왜* 인지는 아래.

---

## 1. cardinality = "서로 다른 값의 개수"
- 저: status(3), bool(2) / 고: user_id·API key·SKU·event_id(UUID) (수백만~전부 고유)

---

## 2. index bloat 란 + 2원인

> **bloat = 인덱스가 live 엔트리에 필요한 것보다 더 많은 8KB 페이지를 차지하는 상태.** (인덱스는 row 가 아니라 `(값, 포인터)` 엔트리를 담음)

| 원인 | 설명 | 연결 |
|------|------|------|
| **A. dead 엔트리** | UPDATE/DELETE → dead tuple → 가리키던 인덱스 엔트리도 dead → VACUUM 전 잔존 | [MVCC](postgresql-mvcc-and-vacuum.md) |
| **B. 반찬 페이지** | 랜덤 키 삽입 → page split 50/50 → 반만 찬 페이지 (UPDATE 없이 INSERT 만으로도) | [B-tree split](postgresql-btree-and-heap-structure.md) |

고-cardinality 가 B 를 키우는 이유: ① 랜덤(UUID)이면 split 폭발 ② 16바이트로 큼 ③ 전부 고유 → **dedup 압축 0**(저-card 는 "값 1번 + 위치 리스트"로 압축).

---

## 3. bloat → "degraded performance" 사슬 (읽기·쓰기 둘 다)

```
인덱스 bloat (2~3배)
  → 캐시(shared_buffers) 초과 → 인덱스 leaf 축출 → 조회마다 디스크 hit (읽기 ↓)
  → INSERT 마다 비대 인덱스 갱신 + split 비용 (쓰기 ↓)
```
→ 빠르려고 만든 인덱스가 **읽기·쓰기 둘 다 배신.**

---

## 4. 읽기 양날 — WHERE 면 자산, GROUP BY 면 폭탄

먼저 반전: **point 조회엔 고-card 가 유리** (`user_id=X` → 1 row 즉시. 저-card bool 은 절반 걸려 인덱스 쓸모없음).

| 고-card 컬럼을… | 결과 | 읽기 |
|---|---|---|
| **WHERE** (`user_id=X`) | 1 row 로 좁힘 | ✅ 빠름 |
| **GROUP BY / DISTINCT** (`per user_id`) | 그룹 수백만 | ❌ 폭발 |

읽기 비용 = ① 몇 row 만지나(WHERE 선택도) ② 몇 그룹 만드나(GROUP BY cardinality).

---

## 5. GROUP BY 내부 — 왜 그룹 수가 비용인가

`SELECT agent_id, SUM(amount) ... GROUP BY agent_id` → PG 의 **Hash Aggregate**:
- **해시 테이블**(그룹당 칸 1개)을 만들고 row 를 훑으며 칸에 누적.
```
{}  → (A,100):{A:100} → (B,50):{A:100,B:50} → (A,200):{A:300,B:50}
    → (C,30):{...C:30} → (B,70):{A:300,B:120,C:30} → 칸 3개 출력
```
- **칸 개수 = GROUP BY 컬럼의 cardinality.**
- agent 3명 → 칸 3개 → `work_mem`(기본 4MB) 안 → 빠름.
- user 500만 → **칸 500만** → work_mem 초과 → **디스크 spill** → 느림.

→ "그룹 수백만" = "해시 칸 수백만 = 메모리 폭발". DISTINCT 도 같은 원리(값마다 칸).

---

## 6. 철학: 고-cardinality 는 "어디에 두느냐" 가 전부

본질 = **값이 잘게 쪼개짐 = 선택도 높음.** 그게 자산이냐 폭탄이냐는 **두는 곳**이 결정:

| 두는 곳 | 고-card 가 | 왜 |
|---|---|---|
| DB 인덱스 (point lookup) | ✅ 자산 | 선택도 = 인덱스 가치 |
| DB 인덱스 (저장/유지) | ⚠️ 비용 | bloat (§2) |
| **메트릭 label (Prometheus)** | ❌ 폭탄 | label 조합당 **시계열 1개** → cardinality explosion |
| **로그 label (Loki)** | ❌ 폭탄 | 조합당 stream 1개 |
| 컬럼형 OLAP (ClickHouse) | ✅ 싸다 | sparse index + 컬럼 압축, B-tree bloat 없음. 집계는 벡터화 + 근사(`uniqHLL`) |

**통합 원리**: 시스템이 **distinct 값마다 물리 산출물 1개**(시계열/stream/B-tree 엔트리/파티션)를 만들면 → 고-card = 폭발. 값을 **데이터로 저장 + sparse 인덱스**(컬럼형)면 → 싸다.

→ **AgentPit 은 이미 이 철학을 코드에 박음**: "Loki high-cardinality label 금지" / "메트릭=Prometheus(저-card label)" / "고-card 식별자(session_id·agent_id·event_id)는 events store 에 **structured field 데이터로**" = Structured-first 원칙의 근거.

---

## 7. 완화
- **시간순 키(UUIDv7)** → 랜덤 삽입을 순차로 → split bloat 차단 ([B-tree](postgresql-btree-and-heap-structure.md), [pk-strategy](pk-strategy-uuid-vs-bigint.md))
- **BRIN**(시간 컬럼) → row 마다 엔트리 없음 → bloat 면역
- 인덱스 절제 / partial index / REINDEX / autovacuum 모니터링
- 진짜 고-card **ad-hoc 필터·집계** → **컬럼형 OLAP** 졸업 (원문이 가리키는 결말)

---

## 한 줄
> 고-card = **WHERE(찾기) 자산, GROUP BY(집계) 폭탄.** 쓰기엔 **index bloat**(dead 엔트리 + 반찬 페이지), 읽기엔 **그룹 폭발**(해시 칸 = cardinality → work_mem spill). 해결은 시간순 키·BRIN·절제, 한계 넘으면 OLAP. 그리고 **메트릭/로그 label 엔 절대 고-card 금지.**

## 관련 노트
- [B-tree & heap 구조](postgresql-btree-and-heap-structure.md) · [MVCC & VACUUM](postgresql-mvcc-and-vacuum.md)
- [row-store vs columnar](row-store-vs-columnar-storage.md) · [OLTP vs OLAP](oltp-vs-olap-and-customer-facing-analytics.md) · [PK 전략](pk-strategy-uuid-vs-bigint.md)
