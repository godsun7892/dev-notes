# PostgreSQL B-tree 와 heap 구조 (페이지·노드·page split)

> "8KB 페이지가 어떻게 인덱스(트리)와 테이블(무더기)을 이루나", 그리고 page split.
> UUID PK *결정/벤치마크* 는 중복 않고 [pk-strategy](pk-strategy-uuid-vs-bigint.md) 참조 — 여기는 *메커니즘*.

---

## 1. 8KB 페이지 = 공통 벽돌

heap(테이블)도 index 도 전부 **8KB 페이지**의 묶음. 데이터를 만지려면 페이지를 [shared_buffers](postgresql-shared-buffers-and-page-lookup.md)로 올림. 단 **둘의 조립 방식이 다름** ↓

---

## 2. heap(테이블) = "무더기" (트리 아님)

```
heap:  [페이지0][페이지1][페이지2]...   ← 순서 X, 트리 X
        row 들이 빈자리에 아무렇게나 박힘
```
- 특정 값 찾기 = 인덱스 없으면 **전체 스캔(seq scan).**
- ★ PG 는 **heap 테이블** — 테이블 자체엔 순서가 없음. (MySQL InnoDB·SQL Server 의 **clustered index**(PK 가 곧 테이블=트리)와 다름. **PG 는 clustered index 없음** — PK 조차 별도 B-tree 가 heap 을 가리킴.)

---

## 3. B-tree(인덱스) = "페이지들의 트리"

```
              [ Root 페이지 ]            ← 8KB 1장 = 노드 1개
             /      |       \
       [내부][내부][내부]               ← 각 8KB 1장
       / | \              \
   [leaf][leaf] ........ [leaf]          ← 실제 (값,포인터) 엔트리
```
- **페이지 = 노드.** 페이지들이 부모→자식 포인터로 연결돼 트리.
- **페이지 *내부* 는 트리가 아니라 정렬된 배열** → 페이지 안 탐색 = **이진 탐색.**
- 두 층 탐색: ① 페이지 사이 = 트리 내려가기(root→leaf, log N) ② 페이지 안 = 정렬 배열 이진 탐색.
- leaf 끼리 **좌↔우 링크** (범위 스캔 `BETWEEN` 시 옆으로 쭉).
- leaf 엔트리 = `(값, TID)`, TID = `heap 페이지#+슬롯` → heap 의 실제 row 를 가리킴.

---

## 4. 캐시 상주 — 모든 인덱스가 메모리에 있는 게 아니다

`shared_buffers` 는 데이터 전체의 **작은 창**. 층마다 운명이 다름:
```
[Root]      ← 1장 + 모든 조회 거침 → 거의 항상 캐시
[내부]      ← 수 적고 자주 거침 → 대체로 캐시
[leaf 수백만]← 너무 많음 → 최근 만진 것만, 대부분 디스크
```
→ 인덱스가 부풀면 leaf 수↑ → **캐시 들어갈 비율↓ → 디스크 hit↑** (bloat 가 아픈 이유).

---

## 5. page split 메커니즘

새 값은 **정렬된 제자리**로. 그 leaf 페이지가:
- **빈칸 있음** → 끼움. 끝. (싸다)
- **꽉 참** → **split**: ① 새 페이지 할당 ② 엔트리 절반 이동 ③ 새 값 삽입 ④ **부모에 새 페이지 포인터 추가**(꽉 찼으면 부모도 split → root 까지 **연쇄**) ⑤ 전부 **WAL**. (비쌈)

---

## 6. 순차 vs 랜덤 — fill density 가 갈린다

| | 순차 키 (bigint/timestamp/UUIDv7) | 랜덤 키 (UUIDv4/해시) |
|---|---|---|
| 삽입 위치 | 항상 **맨 오른쪽 leaf** | 트리 **전역 아무 데나** |
| split | rightmost (왼쪽 ≈90% 유지) | 중간 **50/50 (반찬)** |
| 평균 fill | ~90% dense | ~50~80% 성김 |
| 캐시 locality | 오른쪽 페이지 hot | 흩어진 페이지 cold → 삽입도 랜덤 I/O |

```
순차: [1,2,3,4]꽉 → 5는 새 오른쪽 페이지   → 왼쪽 전부 꽉
랜덤: [a,c,e,g]꽉 + d삽입 → split → [a,c][d,e,g]  ← 둘 다 반만 참
```
→ 같은 row 수에 랜덤은 **페이지 1.3~2배** = bloat. (실측 leaf fill: bigint 97% / v7 90% / v4 79% — [pk-strategy](pk-strategy-uuid-vs-bigint.md))

> UUID v4 vs v7 vs bigint **선택/벤치마크/변곡점**은 [pk-strategy](pk-strategy-uuid-vs-bigint.md)에. 여기선 "왜 그런가(split)"만.

---

## AgentPit 메모
- `deterministic event_id`(sha256→UUIDv8)는 **해시라 사실상 랜덤** → events PK B-tree 가 §6 랜덤 패턴(반찬 bloat) 위험. 고-ingest events → **시간 prefix 키(v7 스타일)** 검토 가치.

## 관련 노트
- [shared_buffers & 페이지 조회](postgresql-shared-buffers-and-page-lookup.md)
- [MVCC & VACUUM](postgresql-mvcc-and-vacuum.md)
- [고-cardinality & index bloat](postgresql-high-cardinality-and-index-bloat.md)
- [PK 전략 UUID vs BIGINT](pk-strategy-uuid-vs-bigint.md)
