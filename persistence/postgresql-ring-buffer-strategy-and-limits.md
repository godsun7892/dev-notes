# PostgreSQL ring buffer (BAS) 와 그 한계 + hint bit

> PG 의 churn 방어책 = ring buffer(Buffer Access Strategy). 그리고 **왜 고객 대면 분석엔 무력한가** 4구멍.
> 선행: [clock-sweep & churn](postgresql-clock-sweep-and-cache-churn.md).

---

## 1. ring buffer 정체

- **Buffer Access Strategy(BAS)**: "대량 읽지만 재사용 안 할" 데이터를 메인 풀에 안 쏟고 **작은 전용 링(슬롯 몇 개)에 가둬 재활용.**
- **별도 메모리 공간이 아님** — `shared_buffers` **안의 격리된 구석.** "이 작업은 이 32칸만 써" 라는 **규칙.**
- 종류: **BAS_BULKREAD 256KB(=32페이지)** — 큰 seq scan(테이블 > shared_buffers/4) / BULKWRITE 16MB / VACUUM 256KB.
- 메커니즘: 32 슬롯 원형 재사용 — 페이지 N 읽고 처리 후 N+32 가 덮어씀. 수백만 페이지가 32칸 통과, **나머지 책상은 안 건드림.** **per-실행**(쿼리 하나마다 자기 링).
- → "큰 테이블 한 방 스캔이 캐시 전체를 날리는" 순진한 사고는 막아줌.

---

## 2. 핵심: ring 여부 = "인덱스냐 본문이냐"가 아니라 "어떻게 읽느냐"

ring 은 **데이터 종류**(인덱스/본문)가 아니라 **읽는 방식**(통독/점프)으로 갈림.

| 작업 | 읽는 것 | ring? |
|------|---------|-------|
| 큰 seq scan (통독) | 본문(heap) 처음→끝 | ✅ ring (구석 32칸) |
| index scan (점프) | ① 인덱스 페이지 ② 본문(heap fetch) | ❌ 둘 다 메인 풀 |

- **같은 본문 페이지**라도 통독으로 읽으면 ring, 색인 점프로 읽으면 메인 풀.
- **인덱스 페이지는 항상 메인 풀** (index scan 은 ring 자체를 안 켬).

---

## 3. 왜 고객 대면 분석엔 무력한가 — 4구멍

| # | 구멍 | 핵심 |
|---|------|------|
| ① | **인덱스 스캔 한정** | 점프는 ring 안 켬 → 인덱스+본문 전부 메인 풀. 작은 테이블 seq scan(<1/4)도 ring 미발동 |
| ② | **동시성** | ring 은 per-쿼리 + **본문만** 가둠. 수천 쿼리 × (인덱스 페이지 + B-tree 상위 + 정렬/해시 상태)가 메인 풀 매몰. 동시성은 churn 증폭기 |
| ③ | **링이 작음(32칸)** | ring 전제 = "한 번 보고 뒤 안 봄". join/sort/집계는 재접근·대량 보유 필요 → 32칸 넘쳐 메인 풀 fallback. (②=양의 문제, ③=구조의 문제. 혼자 돌아도 샘) |
| ④ | **dirty/hint bit** | fresh 데이터를 *읽기만* 해도 페이지 dirty → 즉시 못 덮어씀 → 메인 풀 누수 (§4) |

> 결론: ring 은 **"한 명의 큰 통독"**(리포트/VACUUM) 전용. 고객 대면 = **수천 동시 + 인덱스 구동 + join/sort + fresh 데이터** → 4구멍으로 다 새서 메인 책상 휩쓺.

---

## 4. hint bit — "읽기인데 쓰기" (④의 핵심)

### 배경: MVCC 가시성
- 각 row 버전(tuple)엔 **xmin**(생성 트랜잭션). 가시성 판단하려면 "xmin commit/abort?" 를 **CLOG**(commit log) 에서 확인 — 매번은 비쌈.

### hint bit = 그 답을 tuple 에 캐싱
- PG 가 한 번 확인하면 답을 **tuple 에 hint bit 도장** → 다음부턴 CLOG 안 봄.
- **도장 찍기 = 페이지 수정 = dirty.**
- → **fresh row 를 처음 읽는 SELECT 가 hint bit 로 페이지 dirty 화** ("읽기인데 쓰기").

### 왜 우리 events 가 표적
- 갓 append 된 fresh 데이터는 hint bit 안 찍힘 → 관전/분석이 대량 스캔 → **수많은 페이지 dirty → 링 슬롯이 메인 풀로 줄줄 누수.**
- (옛 데이터는 이미 도장 → 재-dirty 안 됨)
- pin(refcount>0) 페이지도 비슷하게 ring 탈출하나 주범은 hint bit.

---

## 관련 노트
- [clock-sweep & cache churn](postgresql-clock-sweep-and-cache-churn.md)
- [row-store vs columnar](row-store-vs-columnar-storage.md) — 분석이 본질적으로 느린 또 다른 이유
- [OLTP vs OLAP 고객 대면 분석](oltp-vs-olap-and-customer-facing-analytics.md)
