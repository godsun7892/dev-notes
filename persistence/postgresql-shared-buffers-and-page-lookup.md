# PostgreSQL shared_buffers 와 페이지 조회 (hit/miss)

> "PG 가 데이터를 *어디에 캐시하고 어떻게 찾는가*" 의 토대 노트.
> 시리즈: 이 노트 → [clock-sweep & churn](postgresql-clock-sweep-and-cache-churn.md) → [ring buffer & 한계](postgresql-ring-buffer-strategy-and-limits.md).
> 큰 맥락은 [OLTP vs OLAP 고객 대면 분석](oltp-vs-olap-and-customer-facing-analytics.md).

---

## 1. shared_buffers — PG 의 내부 메모리 캐시

- PG 는 자기만의 **공유 메모리 페이지 캐시 = `shared_buffers`**. **8KB 페이지(block)** 를 담는 슬롯 배열. 기본 128MB, 보통 **RAM 25%**.
- **모든 것이 8KB 페이지** — 테이블(heap) 도, **인덱스도** 디스크에 8KB 페이지로 저장. 데이터를 만지려면 그 페이지를 먼저 shared_buffers 슬롯으로 올려야 함.
- → **인덱스가 별도 특별한 게 아님.** 인덱스도 8KB 페이지라 같은 공용 캐시에 올라와 같은 규칙으로 경쟁.

도서관 비유: **디스크 = 거대한 서고**(느림), **shared_buffers = 책상**(작지만 손 뻗으면 바로). 읽기 = "서고에서 책상으로 올림".

---

## 2. 캐시는 2층이다 (double buffering)

PG 는 기본 buffered I/O → 캐시가 두 겹:

| 층 | 무엇 | 속도 |
|----|------|------|
| L1 `shared_buffers` | PG 자기 캐시 | 가장 빠름 (포인터 반환, µs) |
| L2 OS page cache | 커널이 든 같은 8KB | shared_buffers miss 여도 커널 hit 면 memcpy (RAM 속도) |
| L3 디스크 | 둘 다 miss | SSD ms, HDD 더 |

→ **`shared_buffers` miss ≠ 디스크 읽기.** OS 캐시가 받쳐줄 때 많음. (관측 시 `blks_read` 가 디스크가 아닐 수 있는 이유)

---

## 3. BufferDesc — 각 슬롯의 포스트잇(메타데이터)

책상에 페이지를 올릴 때 자리마다 작은 포스트잇이 붙음:

| 항목 | 도서관 비유 | 의미 |
|------|------------|------|
| **tag** | "이 자리에 어느 책 몇 페이지" | 이 슬롯이 **어느 페이지**인지 = `(relfilenode, fork, block번호)` |
| **usage_count** | 최근 몇 번 봤나 정(正)자 | 인기도 (0~5). 접근 시 +1 |
| **refcount** | 지금 이 책 보는 사람 수 | pin. >0 이면 **축출 금지** |
| **dirty** | "연필로 고쳐 적음" | 수정됨. 디스크 저장 전엔 **못 버림** |

이 포스트잇으로 "책상 꽉 차면 누구를 돌려보낼지" 판단 (→ [clock-sweep](postgresql-clock-sweep-and-cache-churn.md)).

---

## 4. tag = 디스크 주소 = 이름표 (한 좌표의 두 얼굴)

- 페이지는 디스크에서 **자리를 안 옮김** → "테이블 X 5번 블록" 은 영원히 같은 디스크 위치(집 주소).
- 그래서 그 좌표(`relfilenode + block번호`)가 **이름표이면서 주소**를 겸함:

| 역할 | 쓰임 |
|------|------|
| **이름표 (조회 키)** | "X-5번이 책상에 있나?" 를 찾는 키 (§5 해시) |
| **주소 (I/O 좌표)** | miss 적재 / dirty 축출 시 **어디서 읽고 쓸지** |

→ "어디 사는지" 와 "이게 뭔지" 가 PG 에선 **같은 좌표.**

---

## 5. buffer table — "책상에 있나?" 를 단번에 (hit/miss 가 갈리는 지점)

- 책상 슬롯 ~16,000개 (128MB ÷ 8KB). 매 접근마다 전수 검색은 불가(초당 수십만 번).
- **buffer table** = `tag → buffer id` **해시 테이블** = 책상 입구의 **색인 카드함.**
- **핵심**: buffer table 은 데이터가 아니라 **"어느 슬롯에 뭐가 있나" 목차.** 카드로 슬롯 번호를 알아낸 뒤 그 슬롯의 페이지를 읽음.

### 흐름
```
"X-5번 줘"
   │ tag 해시
   ▼
[buffer table] ── 카드 있나?
   ├ HIT  → 슬롯 pin(refcount++) + usage_count++ → 포인터 (µs)
   └ MISS → victim 확보(clock-sweep) → dirty 면 먼저 write
            → 디스크/OS 적재 → 카드 새로 등록 → pin, usage_count=1 (ms)
```

- 동시성: 공유 해시라 **128개 partition lock** 으로 경합 분산 (한 명이 카드 만질 때 전체 안 멈춤).

---

## AgentPit 메모

- **hot working set** = 활성 세션 ERP row + hot 인덱스 상위 페이지. 평소 L1 상주 → 게임 트랜잭션 µs.
- 이 hot set 이 분석 부하에 밀려나면 churn (→ [다음 노트](postgresql-clock-sweep-and-cache-churn.md)).

## 관련 노트
- [clock-sweep & cache churn](postgresql-clock-sweep-and-cache-churn.md)
- [ring buffer strategy & limits](postgresql-ring-buffer-strategy-and-limits.md)
- [PostgreSQL locking 심층](postgresql-locking-deep-dive.md)
