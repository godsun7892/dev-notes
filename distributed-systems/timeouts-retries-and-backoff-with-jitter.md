# Timeouts, Retries, and Backoff with Jitter

| 항목 | 내용 |
|------|------|
| 출처 | https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/ |
| 저자 | Marc Brooker — AWS Senior Principal Engineer |
| 주제 | 분산 시스템에서 timeout/retry/backoff 를 안전하게 다루는 방법 |
| 보조 자료 | Jeff Dean & Luiz Barroso, "The Tail at Scale" (CACM 2013) |

> 분산 시스템 안정성 운영의 고전 essay. 권고가 짧지만 각 권고가 "왜" 라는 근거를 명확히 제시. retry/timeout 을 "프레임워크가 알아서 해주는 것" 으로 두는 단계에서, 명시적 design 으로 끌어올리는 분기점이 되는 글.

---

## 0. 본문을 풀려면 먼저 잡아야 하는 보조 개념

### 0.1 자원 점유 모델 — 요청은 왜 자원을 점유하는가

소프트웨어는 물리 기계 위에서 도는 시간 의존 프로세스다. "요청을 처리한다" = CPU 가 사이클을 태우고, 메모리에 데이터를 들고 있고, NIC 가 비트를 전송하는 **물리 활동**. 활동에는 시간이 걸리고, 그 시간 동안 자원은 다른 요청에 못 쓰인다.

자원의 종류 — layer 별:

| Layer | 자원 |
|---|---|
| OS | CPU cycle, CPU cache (L1/L2/L3), RAM, FD, kernel socket buffer, ephemeral port, 디스크 IOPS |
| 런타임 | async task slot, thread pool slot, semaphore slot, GC pressure |
| 연결 풀 | DB connection, Redis connection, HTTP client connection |
| 다운스트림 | DB lock, transaction ID, buffer pool, 외부 API rate limit, **돈** (per-call cost) |
| 논리 | latency (사용자 대기 시간 그 자체), 큐 슬롯, 캐시 공간 |

비명시적이라 놓치기 쉬운 4 가지 — **본문 전체가 이 4 개를 전제로 함**:

1. **자원 보유 기간이 다 다르다** — CPU = μs, 메모리 = 요청 기간, DB connection = 트랜잭션 기간, FD = 연결 lifetime. "자원" 으로 한 덩어리로 묶으면 안 됨
2. **자원은 leak 된다** — timeout 안 걸린 hung request 는 자원을 영구 점유. retry storm 의 진짜 무서움은 retry 자체가 아니라 *죽지 않은 원본 + retry 가 자원을 둘 다 들고 있는 것*
3. **자원은 cascading 점유된다** — 요청이 stack 을 타고 내려가면서 layer 마다 자원을 잡는다. upstream 1 개 hang = downstream DB connection 1 개도 hang. **장애 전파 메커니즘이 이것**
4. **물리 자원에는 큐가 자동 생긴다** — 공급 < 수요면 OS / DB lock / TCP backlog 가 묵시적 큐를 만든다. "큐가 없는 시스템" 같은 건 없고, 안 보이는 곳에 있을 뿐

→ 본문의 모든 권고를 한 원리로 압축하면: **유한 자원의 점유 시간을 최소화하고, 점유 중인 자원을 회복 가능한 상태로 유지하라.**

### 0.2 Percentile — p50 / p99 / p99.9

`p` = percentile (백분위수).
- p50 = 50% 가 이 값 이하 = 중앙값
- p99 = 99% 가 이 값 이하. 1% (100 건 중 1 건) 가 이보다 느림
- p99.9 = 99.9% 가 이 값 이하. 0.1% (1000 건 중 1 건) 가 이보다 느림

**평균을 안 쓰는 이유** — 사용자는 평균을 경험하지 않음. 각 사용자는 자기 요청 1 개의 latency 만 경험. 평균은 outlier 에 휘둘리고, "평균 빠른데 가끔 멈춤" 이 "항상 좀 느림" 보다 UX 가 훨씬 나쁜데 평균은 둘을 구분 못 함.

→ p50 = 전형적 사용자, p99 = 운 나쁜 사용자, p99.9 = 시스템의 진짜 약점.

### 0.3 Fanout tail amplification — "백엔드의 p99 가 사용자의 p50 이 된다"

요청 1 개가 백엔드 N 개에 fanout 하고 모든 응답을 기다린다면, 사용자 요청이 "빠르다" = **N 개가 전부 빠르다** 일 때만 성립.

각 백엔드가 99% 확률로 빠르다고 가정:

| 백엔드 fanout N | P(모두 빠름) = 0.99^N | P(사용자가 느림) |
|---|---|---|
| 1 | 99% | 1% |
| 10 | 90.4% | 9.6% |
| 50 | 60.5% | 39.5% |
| 100 | 36.6% | **63.4%** |
| 230 | ~10% | ~90% |
| 500 | 0.7% | 99.3% |

**백엔드의 1% 사건이 N=100 fanout 에선 사용자의 63% 사건**. 통계 용어로는 max-of-N 분포의 평행이동.

직관: 100 명에게 "1% 확률로 손드세요" 시키면 한 명도 안 들 확률이 36% 밖에 안 된다. 그 1 명이 전체를 결정.

→ 출처: Dean & Barroso, "The Tail at Scale" (2013). 분산 시스템에서 **"백엔드 잘게 쪼개기" 가 항상 좋은 게 아닌 이유**의 수학적 근거.

→ 처방: fanout N 줄이기 / 꼬리 자르기 (공격적 timeout) / hedged request / tail-tolerant 설계.

---

## 1. Timeout 선택

### 1.1 Downstream latency metric 에서 시작한다

핵심 명제:

> **timeout 은 호출자 (caller) 의 추측이 아니라 호출 대상 (callee) 의 통계에서 출발한다.**

실무에서 흔한 안티패턴:
- "프레임워크 기본값이 30s 니까 30s"
- "다른 서비스도 5s 쓰니까 5s"
- "사용자가 못 기다리니까 1s"
- "왠지 충분해 보이니까 60s"

→ 모두 **호출 대상의 실제 동작과 무관**. 데이터 기반 timeout 의 출발은 다운스트림 서비스의 latency histogram 을 보고 p99.9 같은 고 percentile 을 baseline 으로 잡는 것.

워크플로우:
1. CloudWatch / Prometheus / Datadog 에서 downstream 의 latency percentile chart 본다 (충분히 긴 윈도우 — 1 주일 이상)
2. p99.9 를 baseline 으로 잡는다
3. 마진 추가 (네트워크 jitter, retry headroom)
4. false timeout 비율 vs hung 자원 점유 사이에서 운영하며 조정

### 1.2 어떤 percentile 을 쓰나

| 후보 | 문제 |
|---|---|
| p50 | 정상 요청 절반이 timeout — 말도 안 됨 |
| p99 | 1% 가 timeout. 1000 RPS 면 초당 10 건 false timeout — 너무 많음 |
| **p99.9** | **0.1% 만 timeout. 정상 꼬리는 살리고 진짜 outlier 만 자름** |
| p99.99 | 거의 모든 정상 요청 살리지만 hung 감지가 너무 늦음 |

→ p99.9 가 표준 출발점. 작업 빈도/비용에 따라 p99 ~ p99.99 사이 조정.

### 1.3 "Intra-region" 단서가 중요한 이유

Brooker 가 "AWS Region 내부 호출" 로 한정한 데에는 이유가 있음:

**Intra-region 의 특성**:
- 네트워크 latency 작고 안정적 (AZ 간 ~1ms, AZ 내 <1ms)
- jitter 무시 가능
- → "downstream latency" 가 그대로 "내가 기다려야 할 시간" 에 근사
- → downstream metric 이 timeout 의 좋은 출발점

**Cross-region / 외부 API 는 다름**:
- 한·미 baseline RTT 100~150ms
- 경로 변동 큼, BGP/peering 이슈로 spike
- → 네트워크 baseline + 변동성 마진 추가 필요

같은 방법론이지만 식이 달라진다.

### 1.4 분산이 좁은 서비스 (예외 케이스 1)

p99.9 timeout 방법의 전제: **"정상 latency 의 꼬리" 와 "비정상/hung" 사이에 의미 있는 gap 이 있다**.

분산이 좁은 서비스 (in-memory cache, 잘 튜닝된 fast service):
```
p50    = 2ms
p99    = 3ms
p99.9  = 4ms
실제 hung = ??
```
- timeout 을 5ms 로 잡으면: GC pause / network microburst 에 false timeout 폭증
- timeout 을 1s 로 잡으면: 빠른 서비스의 장점 무효화

**자연적 노이즈 크기가 (p99.9 - p50) 보다 크면 p99.9 기준이 무너진다.**

처방:
- SLA / 계약 기반 timeout
- 절대값 timeout
- 상위 layer 신호 (circuit breaker, health check) 와 결합

### 1.5 Bimodal latency — 메트릭 편향 (예외 케이스 2, 더 깊은 함정)

실제 incident: 어떤 팀이 metric 의 p99.9 = 20ms 보고 timeout 을 20ms 로 잡음. 평소 잘 작동. **배포 직후마다 일부 요청 실패**.

조사 결과:
- 평소 요청은 이미 수립된 connection 재사용 (warm)
- 새 서버 instance 가 뜨면 cold — TCP handshake (1 RTT) + TLS handshake (1~2 RTT) + 인증서 검증 → 20ms 초과
- 측정된 p99.9 는 **steady-state 분포의 p99.9**, cold 모드는 underrepresent

실제 분포는 둘:
```
warm-connection (99.9%)        cold-connection (0.1%, 배포 시 폭증)
    p50    = 5ms                    p50    = 30ms
    p99.9  = 20ms  ← 측정됨          p99.9  = 80ms  ← 안 보였던 것
```

**bimodal latency** — 두 mode 가 섞여있는데 한쪽이 너무 드물어서 평균/p99 에 묻힘. 그 mode 가 활성화되는 사건 (배포 / scale-out / connection 재활용) 이 일어나면 SLA 깨짐.

같은 구조의 반복 패턴:

| 평소 모드 | 사건 모드 (메트릭에 안 보임) |
|---|---|
| 캐시 hit | 캐시 miss / cold |
| warm DB connection | new connection / pool exhausted |
| in-memory data | disk read 필요 |
| local replica | failover 후 cross-region |
| JIT compiled | cold path |

처방:
- timeout 을 layer 별로 분리 (`connect_timeout` vs `request_timeout`)
- 연결 pre-warming
- 메트릭에 `connection_age` / `is_first_request` dimension 추가
- mode-aware timeout

**메타 교훈**: 메트릭이 무엇을 측정한 것인지 의식하지 않으면, 같은 방법론을 따라도 사고가 난다.

---

## 2. Timeout 구현의 함정

### 2.1 `SO_RCVTIMEO` 는 end-to-end 가 아니다

Linux 소켓 옵션. `setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, ...)`.

역할: `recv()` **1 회**가 블록될 수 있는 최대 시간. 5초 설정하면 recv() 가 5초 동안 데이터 없으면 `EAGAIN` 반환.

결정적 결함 — **타이머가 매 recv() 마다 리셋된다**.

시나리오: 10MB 응답을 읽음. `SO_RCVTIMEO = 5s`. 서버가 1초마다 1KB 흘려보냄:
```
t=0    recv() 호출
t=1s   1KB 도착 → recv() 성공 (타이머 리셋)
t=1s   recv() 다시 호출
t=2s   1KB 도착 → recv() 성공 (타이머 리셋)
...
t=10000s 전부 받음 — 타임아웃 안 걸림
```

각 `recv()` 는 5초 안에 받았으니 timeout 안 걸림. **전체는 거의 3시간**. 5초 timeout 이 아무것도 보호 못 함.

이게 **Slowloris 공격**의 원리이기도 함. 공격자가 (SO_RCVTIMEO - 1) 간격으로 1 바이트씩 보내면 연결이 영원히 안 끊긴다.

→ **`SO_RCVTIMEO` 는 "바이트 간 간격" 의 상한이지 "전체 응답 시간" 의 상한이 아니다.**

### 2.2 언어별 차이

| 언어 | primitive | end-to-end timeout |
|---|---|---|
| C / Linux | `SO_RCVTIMEO` | ❌ |
| **Java** | `Socket.setSoTimeout()` (= SO_RCVTIMEO wrap) | ❌ — Brooker 가 지적 |
| **Go** | `conn.SetDeadline(time)` — 절대 시각 | ✅ |
| Python asyncio | `asyncio.timeout()` / `asyncio.wait_for()` | ✅ |

**Go 의 `SetDeadline` 이 다른 점**:
```go
conn.SetDeadline(time.Now().Add(5 * time.Second))
// 5초 후 시각이 절대 deadline.
// 4.9s 에 recv() 성공해도 5.0s 의 다음 recv() 는 즉시 실패.
// 진행 중인 recv() 도 5.0s 에 강제 중단.
```

구현: Go 런타임이 epoll timeout 으로 deadline 까지 남은 시간을 매번 새로 넘긴다. 즉 `SO_RCVTIMEO` 안 쓰고 **사용자 공간에서 절대 deadline 을 직접 관리**. `context.Context` 와 결합되면 함수 호출 chain 전체에 deadline 전파.

### 2.3 메타 통찰

> **"timeout 5s" 라고만 말하는 건 의미가 없다. 어떤 timeout 인지 명시해야 한다.**

종류:
- connect timeout (TCP handshake)
- TLS handshake timeout
- per-write / per-read timeout (= SO_RCVTIMEO)
- end-to-end / request timeout
- idle timeout

`SO_RCVTIMEO` 만 잡고 "타임아웃 처리 끝" 이라 생각하면 위 시나리오에서 죽음. **end-to-end deadline 이 별도로 있어야 함.**

abstraction leak 의 사례 — OS / 언어 / 라이브러리 / 코드 사이에서 timeout 의 의미가 변형되는데, 그걸 의식하지 못하면 "권고를 코드로 옮길 때" 실패함.

---

## 3. Retry 의 amplification

### 3.1 곱셈 증폭

각 layer 가 독립적으로 retry 하면 부하가 **곱셈**으로 증폭:

```
사용자 → API gateway → service A → service B → service C → DB
            3 회         3 회         3 회        3 회       3 회

1 사용자 요청 = 3 × 3 × 3 × 3 × 3 = 3^5 = 243 회 DB 호출
```

DB 가 잠깐 느려져서 retry 가 trigger 되는 순간 → 회복하려는 DB 에 **243 배 부하** → 회복 불가능. 이게 retry storm / retry amplification.

### 3.2 한 layer 에서만 retry

권고: **여러 layer 가 동시에 retry 하면 안 된다. 한 지점에서만.**

어느 layer 인가는 trade-off:

| Retry 위치 | 장점 | 단점 |
|---|---|---|
| 낮은 layer (DB 호출 근처) | 작은 단위만 다시 함 → 낭비 적음 | retry burden 이 가장 약한 layer 에 집중 |
| 높은 layer (사용자 진입점) | 사용자 context 있음, idempotency 관리 쉬움 | 중간 layer 의 일이 다 낭비됨 |

**Amazon 의 default**: 가장 높은 layer 에서 retry, 아래 layer 들은 retry 끔. 이유:
- 낮은 layer 의 일시 실패는 어차피 빠르게 회복되는 경우 많음
- 진짜 장애면 낮은 layer retry 도 어차피 실패
- 높은 layer 가 idempotency / user context / 비즈니스 의미 다 가짐 → smart retry 가능

운영 측면:
- 명시적 정책 결정 (design doc 에 "retry 는 X layer 가 책임" 박음)
- 라이브러리 default retry 끄기 (AWS SDK, gRPC client, HTTP client 들이 자동 retry)
- retry 발생 시 metric 으로 가시화

---

## 4. Retry 제한

retry 를 한 layer 에서만 한다고 해도 **그 layer 에서도 무한 retry 금지**. 어떻게 제한하나? 두 후보:

1. **Circuit breaker** (Netflix Hystrix 로 산업 표준)
2. **Token bucket** rate limiting (Amazon 의 선호)

### 4.1 Circuit breaker 와 modal behavior 문제

3-상태 머신:
```
        실패율 임계 초과
CLOSED  ─────────────→  OPEN  (모든 호출 즉시 실패)
  ↑                       │
  │ 복구 확인              │ cooldown 경과
  │                       ↓
  └──── HALF_OPEN ←──────┘  (probe 몇 개로 시험)
```

직관적으로 좋아보임 — "죽은 거 같으면 그만 두드려서 회복할 시간 주자". 그러나 Brooker 의 비판:

**(1) Modal behavior 가 테스트하기 어렵다**
- 각 mode 단독 테스트는 쉬움
- mode 간 전환 로직이 어려움 — race condition, timing, 임계값 boundary
- HALF_OPEN 에서 probe 2 개 성공, 1 개 실패면? → 미묘
- 가장 큰 문제: OPEN mode 는 실제 운영에서 거의 발동 안 됨 → 테스트 덜 됨 → 실전에서 처음 작동할 때 버그

**(2) 복구 시간을 크게 늘린다**

```
t=85s    downstream 다운, circuit OPEN 진입
t=100s   downstream 실제 회복
t=100~145s  너의 circuit 은 아직 OPEN → 호출 0 회 (downstream 멀쩡한데 못 씀)
t=145s   HALF_OPEN, probe 시작
t=146s   probe 성공 → CLOSED 복귀
t=146s   그동안 쌓였던 retry 들이 일제히 폭발 → downstream 재차 다운 → 다시 OPEN
```

문제:
- downstream 회복 후 ~45초 동안 인위적으로 안 부름 → 복구가 cooldown 만큼 지연
- 갑작스러운 복귀 시 밀린 요청 폭발 → 재차 다운 → cycle 무한반복

→ Circuit breaker 는 **binary 사고** ("죽었다 / 살았다"). 현실은 grayscale ("capacity 가 줄었다, 일부는 처리 가능") 인데 binary 로 over-react.

### 4.2 Token bucket 의 연속 rate 제한

개념:
- 양동이에 token N 개 (예: 100)
- 초당 R 개 (예: 5/sec) 자동 refill
- retry 1 회 = token 1 개 소비
- token 있으면 retry OK, 없으면 retry 안 함 (또는 refill 속도에만 맞춰서)

작동 그림:
```
정상              : token 100/100, 마음대로 retry
일시 장애         : retry 폭증 → token 빠르게 소진
지속 장애         : 5/sec 만 retry (refill 속도) → downstream 보호
downstream 회복   : retry 성공률↑ → 자연스럽게 retry 줄어듦 → token 차오름
```

**결정적 차이 — Mode 가 없다.** 항상 단일 코드 경로:
```python
if bucket.try_consume(1):
    do_retry()
else:
    pass  # skip retry
```

"OPEN 상태" 같은 것 없음. 변화는 부드럽고 점진적. 복구도 자연스러움 — downstream 회복 즉시 따라가고, 폭발적 재진입 없음 (bucket size 가 burst 한도).

### 4.3 비교

| | Circuit breaker | Token bucket |
|---|---|---|
| 사고 방식 | binary | 연속 (rate 조절) |
| Mode 개수 | 3 | 1 |
| 테스트 난이도 | 어려움 (전환, race) | 쉬움 |
| Downstream 회복 시 | cooldown 만큼 지연 | 즉시 따라감 |
| 재진입 폭발 | 자주 발생 | 거의 없음 |
| 보호 강도 | 100% (OPEN 시 0 호출) | 비례적 |
| 코드 복잡도 | 상태 머신 | 단순 카운터 |

### 4.4 메타 통찰

> **"Modal behavior 는 거대 분산 시스템에서 항상 패배한다."**

이유:
- 거의 안 발동하는 mode 는 거의 안 테스트됨
- 사고 한가운데서 처음 발동하는 코드는 거의 항상 버그
- 사고는 사람들 panic 상태에서 일어남 → modal 시스템은 디버깅 더 어려움
- 회복 중 시스템이 모드 전환하면 회복이 오히려 지연됨

→ **운영의 단순함 > 이론적 우아함**. token bucket 은 항상 같은 일을 하고 그 강도만 변함.

(Circuit breaker 가 의미 있는 경우도 있음: downstream 이 확실히 죽었고 호출 자체가 추가 부하만 되는 경우 / 사용자에게 빨리 에러 보여야 할 때 / fallback path 가 있을 때. 다만 retry 제한 용도로는 token bucket 이 더 안전한 default.)

---

## 5. Idempotency — Retry 의 전제조건

지금까지의 권고 (timeout / retry / token bucket) 가 모두 **idempotent operation 일 때만 성립**. 안 갖춰지면 위 모든 권고가 무의미.

### 5.1 Side effect 와 retry 위험

API 호출이 호출 자체 바깥 세상을 바꾸는가:

| API | side effect? |
|---|---|
| `GET /user/123` | ❌ |
| `POST /payments` | ✅ 카드 결제 |
| `POST /users` | ✅ 생성 |
| `DELETE /user/123` | ✅ 삭제 |
| `EC2 RunInstances` | ✅ VM 시작 (돈 나감) |

핵심 시나리오:
```
client                            server
  │                                  │
  │── POST /payments ──────────────→ │
  │                                  │ 처리: $100 결제 ✓
  │                                  │ 응답 생성
  │ ← 응답이 네트워크에서 사라짐 ✗ ─│
  │                                  │
  │ (timeout)                        │
  │ "처리됐나? 안 됐나? 모름"           │
  │                                  │
  │── POST /payments (retry) ──────→ │
  │                                  │ 처리: $100 다시 결제 ✓✓
  │                                  │
  │ "성공!" (사실은 $200 결제됨)        │
```

**근본 문제**: client 는 "서버에 도달 안 함" vs "서버 처리 후 응답 사라짐" 을 구분 못 함. 둘 다 timeout / connection error 로 보임. 안전 retry 의 유일한 방법 = **서버가 중복을 인식하고 한 번만 처리**.

### 5.2 Idempotency 정확한 정의

수학: `f(f(x)) = f(x)`. N 번 적용 = 1 번 적용.

API: **N 번 호출 = 1 번 호출과 같은 최종 상태**.

| operation | idempotent? | 이유 |
|---|---|---|
| "X 의 email 을 Y 로 set" | ✅ | N 번 set 해도 email = Y |
| "X 의 credit 1 증가" | ❌ | 매번 +1 누적 |
| "X 삭제" | ✅ | 첫 호출 후 X 없음, 그 다음은 no-op |
| "user 생성 (email=Y)" | ❌ (naive) | 매번 새 user |
| "user 생성 (id=Z, email=Y)" | ✅ | id 충돌로 두 번째 거부 |

핵심: **state 가 결과를 정의** vs **operation 자체가 결과를 정의**. 전자는 idempotent.

읽기 전용 = 보통 idempotent (GET 은 state 안 바꿈).
리소스 생성 = 보통 not idempotent. 이걸 idempotent 로 만드는 게 token 패턴.

### 5.3 Token-based idempotency (EC2 RunInstances 예시)

**Client**:
```python
token = uuid.uuid4()  # logical operation 마다 1 번 생성
for attempt in range(3):
    try:
        result = ec2.run_instances(
            ImageId="ami-...",
            ClientToken=token,  # ← retry 전체에서 SAME token
        )
        break
    except (Timeout, NetworkError):
        continue
```

**Server (EC2 내부)**:
```python
def run_instances(ClientToken, ImageId, ...):
    existing = dedupe_store.get(ClientToken)
    if existing:
        return existing  # 이전 결과 그대로, 새 instance 안 만듦
    instance = launch_instance(ImageId, ...)
    dedupe_store.put(ClientToken, instance, ttl=24h)
    return instance
```

결과: 같은 token 으로 N 번 호출 = instance 1 개.

산업 사례:
- **Stripe**: `Idempotency-Key` 헤더
- **PayPal**: `PayPal-Request-Id` 헤더
- **AWS EC2**: `ClientToken` 파라미터
- **Square**: `idempotency_key` body 필드

### 5.4 Client 구현의 함정 (자주 망함)

```python
# ❌ WRONG — 매 retry 마다 새 token
for attempt in range(3):
    token = uuid.uuid4()
    api.create_payment(token=token, ...)
# 결과: token 매번 달라서 dedupe 무효

# ✅ RIGHT — token 1 번 생성, retry 공유
token = uuid.uuid4()
for attempt in range(3):
    api.create_payment(token=token, ...)
```

원칙: **token 은 "logical operation" 의 정체성**이지 "HTTP 호출 1 회" 의 정체성이 아니다.

자주 망하는 경우:
- retry 로직이 inner 함수에 숨어있어서 outer 가 token 책임 모름
- HTTP client 의 자동 retry 가 끼어있어서 request 재생성 시 token 도 새로 생성

### 5.5 Server 구현의 함정 (사실 더 어려움)

dedupe storage 조건:

**(1) side effect 와 원자적**
```python
# ❌ WRONG — 분리된 트랜잭션
def run_instances(token, ...):
    instance = launch_instance(...)    # 1 단계: side effect
    dedupe_store.put(token, instance)  # 2 단계: 기록
    # 1 후 2 전 crash → instance 만들어졌는데 기록 안 됨 → retry 시 중복

# ✅ RIGHT — 원자적
def run_instances(token, ...):
    with transaction():
        if not dedupe_store.try_insert(token):
            return dedupe_store.get(token)
        instance = launch_instance(...)
        dedupe_store.update(token, result=instance)
```

**(2) Race condition** — 동일 token 동시 호출 2 개
- 둘 다 조회 → 없음 → 둘 다 처리 → 중복
- DB unique constraint / DynamoDB conditional write / Redis SETNX 같은 원자 연산 필수

**(3) TTL** — 너무 짧으면 늦은 retry 시 dedupe 실패, 너무 길면 storage 비용. 보통 24h~7d.

**(4) Side effect 가 외부 시스템**
- 결제 = 외부 PG 사 호출. 내 dedupe 조회 → PG 호출 → dedupe 기록. 중간 crash 시 처리 어려움
- 해결: PG 사 자체가 idempotency token 받음 (cascading idempotency)

### 5.6 다른 idempotency 패턴

token 만 있는 게 아님:

1. **자연키 dedup** — `INSERT ... ON CONFLICT (email) DO NOTHING`. 비즈니스 자체 unique key 활용
2. **조건부 업데이트** — "status 가 pending 일 때만 paid 로 set". 낙관적 락 (version column)
3. **State machine 기반** — "X 상태로 transition". 이미 X 면 no-op
4. **Deterministic event ID** — 입력 기반 deterministic ID (sha256 등) + ON CONFLICT

---

## 6. 에러 코드 dispatch 의 한계

### 6.1 HTTP 4xx vs 5xx 표준 규약

**4xx — Client Error**: "요청 자체가 잘못. 똑같이 다시 보내도 똑같이 실패."
**5xx — Server Error**: "내 잘못. 잠깐 후 다시 시도하면 다른 결과 가능."

표준 retry 정책: 4xx 안 함, 5xx backoff 후 retry. 단순 시스템에선 깔끔.

### 6.2 Eventual consistency 가 경계를 흐림

분산 시스템에서: 상태 전파에 시간이 걸림. 그 동안 다른 부분은 다른 사실을 본다.

**시나리오 1: Read-after-write**
```
t=0.00s   Primary 에 user X 생성 → 200
t=0.05s   Read replica 에서 GET /user/X → 404 (replication 미도착)
t=0.50s   Replication 완료
t=0.51s   GET /user/X → 200
```
404 인데 사실 client error 가 아님. 잠깐 후 retry 하면 성공.

**시나리오 2: IAM 권한 전파 (AWS 의 유명 사례)**
```
t=0     User U 에 S3 read 권한 부여 → 200
t=0.1s  User U 가 S3 GetObject → 403
        (IAM 변경이 enforcement point 에 전파 중)
t=10s   같은 호출 → 200
```
AWS 공식 문서에 명시: "IAM is eventually consistent". 403 인데 retry 해야 함.

**시나리오 3: 분산 캐시 / DNS / cross-region** — 같은 구조 반복.

공통 패턴:
1. 상태가 시스템 어딘가에 분명히 존재
2. 너의 요청이 도달한 지점에는 아직 전파 안 됨
3. HTTP layer 가 "없다 / 안 됨" 으로 보고 4xx 반환
4. 잠깐 후 같은 요청이 성공

→ "client error" 의 의미가 무너짐.

### 6.3 nuanced retry 정책

| Code | 보통 의미 | 분산 환경에서 retry? |
|---|---|---|
| 400 | malformed request | ❌ 절대 안 함 |
| 401 | 인증 실패 | ❌ (credential refresh 별도) |
| 403 | 권한 없음 | ⚠️ 상황 따라 (IAM 전파 race) |
| 404 | 리소스 없음 | ⚠️ read-after-write 면 함 |
| 409 | 충돌 | ⚠️ transient lock 이면 함 |
| 422 | 의미적 오류 | ❌ |
| 429 | rate limit | ✅ backoff 후 |
| 5xx | 서버 오류 | ✅ |

→ status code 만으로 dispatch 하면 안 됨. **operation semantics + 구체적 error type** 까지 봐야 함.

운영 권고:
- 호출하는 API 의 consistency model 알기 (strong? eventual? 어떤 작업이 race window?)
- SDK 가 제공하는 구체적 exception 으로 판단 (status code 가 아니라)
- 짧고 한정된 retry (수렴 안 하면 진짜 에러 — 영원히 retry 금지)

### 6.4 메타 통찰

> **HTTP status code 는 stateless request-response 모델을 가정해 설계됐다. 분산 시스템의 "지금은 안 보이지만 곧 보일 거" 라는 상태는 표현 못 한다.**

같은 종류 leak 이 gRPC status / SQL SQLSTATE / OS errno 에도 반복됨 — abstraction 의 leak 을 인식 못 하면 "표준 따랐는데 왜 사고나지?" 가 됨.

---

## 글 전체 요약

권고 5 단:

1. **timeout** — downstream metric (p99.9) 에서 시작, intra-region 베이스라인, 분산 좁은 서비스 / bimodal 함정 의식
2. **timeout 구현** — `SO_RCVTIMEO` 는 end-to-end 가 아니다. 언어별 primitive 차이 인식, 진짜 deadline 은 별도 layer
3. **retry placement** — 한 layer 에서만 (보통 가장 높은 layer). 다른 layer 의 default retry 끄기
4. **retry 제한** — circuit breaker 의 modal behavior 피하고 token bucket 의 연속 rate 제한 선호
5. **idempotency** — 위 모든 권고의 전제. token 기반 / 자연키 / 조건부 update / deterministic ID 중 선택
6. **error dispatch** — HTTP status code 표준 규약은 분산 환경에서 leak. operation semantics 까지 봐야 함

전체 통합 원리:

> **유한 자원의 점유 시간을 최소화하고, 점유 중인 자원을 회복 가능한 상태로 유지하라.**

운영 메타 원리:

> **운영의 단순함 > 이론적 우아함. modal behavior 는 거대 분산 시스템에서 항상 패배한다.**

---

## 추가로 참고할 만한 출처

- AWS Builders' Library 의 동저자 글들 (Brooker / Colm MacCárthaigh / Jacob Gabrielson / David Yanacek):
  - "Avoiding fallback in distributed systems" (Gabrielson)
  - "Workload isolation using shuffle sharding" (MacCárthaigh)
  - "Reliability, constant work, and a good cup of coffee" (MacCárthaigh)
  - "Using load shedding to avoid overload" (Yanacek)
  - "Avoiding insurmountable queue backlogs" (Yanacek)
  - "Implementing health checks" (Yanacek)
  - "Making retries safe with idempotent APIs" (Featonby)
- Dean & Barroso, "The Tail at Scale", CACM 56(2), 2013 — fanout tail amplification 의 canonical paper
- AWS Architecture Blog, "Exponential Backoff And Jitter" — Brooker 본인의 초기 블로그 글 (Builders' Library essay 의 short form)
- Marc Brooker 블로그 (https://brooker.co.za) — 분산 시스템 trade-off 의 현역 best blogger
