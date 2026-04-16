# 능동 액터와 시간 기반 상태 갱신의 충돌

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 액터(사용자/에이전트)가 자기 상태를 자유롭게 읽고 쓰는 동안, 시간이 흐름에 따라 자동으로 변해야 하는 것들이 같은 데이터를 건드릴 때 어떻게 일관성과 성능을 모두 잡는가? |
| 관련 | `db-time-patterns.md` (저장 측면) |

---

## 1. 문제 정의

많은 시뮬레이션/게임/MMO 시스템에서 한 액터(actor)의 상태는 두 주체가 동시에 건드린다:

```
[액터의 능동 행동]                       [시간이 지나면 자동으로]
- 거래 / 행동 → 재고/잔고 변경            - 작업 완료 → 산출물 추가
- 자원 사용 → 인벤토리 차감              - 자원 만료 → 배치 제거
- 액션 입력 → 상태 갱신                  - 자동 생성/회복 → 자원 추가
```

**둘 다 같은 inventory/balance를 건드린다.** 두 주체가 동시에 접근하면:

- **Race condition**: 액터가 인벤토리 5개로 거래 검증하는 사이, 만료 처리가 3개를 빼가면 검증 결과가 깨짐
- **Lost update**: 자동 갱신이 새 자원을 추가하는 동안 액터가 인벤토리를 읽고 쓰면 추가분이 덮어쓰여 사라짐
- **Phantom read**: 액터가 step N에서 조회 후 의사결정하는데, 그 사이 step N+1로 넘어가면서 일부 자원이 만료됨 → 잘못된 결정

---

## 2. 후보 해결책들

### 2.1 Global Ticker (가장 단순, 가장 나쁨)

매 step마다 중앙 서버가 모든 액터의 상태를 일괄 갱신.

```python
def step_tick(step_number):
    for actor in all_actors:
        complete_jobs(actor)
        expire_batches(actor)
        run_auto_generation(actor)
    # 그 다음에 액터 행동 처리
```

- ❌ 액터 수가 늘면 처리 폭발 (10만 액터 × 매 step)
- ❌ 휴면 액터도 처리 비용 발생
- ❌ Tick 도중엔 액터 행동을 막아야 함 (글로벌 락)
- ❌ 분산 환경에서 글로벌 락 = 확장성 0

### 2.2 Lock + Global Ticker

위의 변형. tick 도중 액터 락을 걸고, 행동은 락 해제 후.

- ❌ 락 경합으로 throughput 폭락
- ❌ 분산 환경에서 분산 락 필요 → 복잡도 폭발
- ❌ 본질적 비효율(휴면 액터 처리)은 그대로

### 2.3 Event-Driven Timer

작업 완료 시점, 만료 시점마다 콜백 등록 → 그 시점에 trigger.

```python
schedule.at(completion_step, lambda: complete_jobs(actor))
schedule.at(expiry_step, lambda: expire_batch(actor, batch))
```

- ✅ 휴면 액터는 콜백만 등록되어 있음 (가벼움)
- ❌ 콜백이 다른 콜백/행동과 충돌 가능
- ❌ 콜백 수가 많아지면 timer wheel 관리 부담
- ❌ 콜백 실행 시점에 액터 행동 중이면 결국 락 필요

### 2.4 Lazy Activation (Orleans Virtual Actor 패턴) ← 권장

**시간은 흐르되, 상태 갱신은 액터가 깨어날 때만 일어난다.**

```python
def activate_actor(state, clock, current_step):
    # 1) 휴면 동안 완료된 작업 처리
    completed = complete_jobs(state, current_step)

    # 2) 휴면 동안의 자동 생성 catch-up (last_wake_step+1 ~ current_step)
    process_auto_generation(state, current_step, clock.last_wake_step)

    # 3) clock 갱신
    clock.last_wake_step = current_step
    return completed
```

핵심 원리:
- **휴면 액터의 상태는 "동결"** — 누구도 안 건드림
- **깨어날 때 한 번에 catch-up** — `last_wake_step`부터 `current_step`까지 일괄 처리
- **catch-up은 같은 worker가 함** — 액터 행동과 같은 컨텍스트에서 실행되므로 동시성 문제 없음
- **만료는 저장하지 않고 lazy 평가** — `expires_at > current_step` 필터로 읽을 때만 판단

---

## 3. 구현 디테일

### 3.1 Worker가 단독 소유

```
[Worker / Task]
   │
   ├── 액터 A를 처리하는 Worker는 액터 A의 상태를 단독 소유
   │
   ├── 같은 step에 액터 A를 처리하는 Worker는 1개만 존재 (배타성)
   │
   └── 다른 Worker가 액터 A의 상태를 동시에 건드릴 일이 없음
```

→ **액터 단위 직렬화**. 분산 락 없이 단일 worker context로 동시성 문제를 우회.

이 패턴은 Orleans의 single-activation 보장과 동일한 원리.

### 3.2 두 단계 catch-up: activate vs tick

```python
# wake 시점 1회 — 휴면에서 깨어남
activate_actor(state, clock, current_step)

# 액터의 작업 loop 도중 step 전진 시
tick_actor(state, clock, current_step)
```

`tick_actor`가 필요한 이유: 액터의 작업 loop이 길어서 (외부 호출, 여러 액션) 진행 중에 글로벌 step 타이머가 넘어갈 수 있음. 그러면 다음 액션 직전에 step이 바뀐 걸 감지하고 1step분 catch-up:

```python
# Worker의 작업 loop
while not done:
    if clock.last_wake_step < current_step:
        tick_actor(state, clock, current_step)

    action = next_action()
    result = execute(action)
```

이렇게 하면 액터가 자기 작업을 하는 도중에도, step 전환에 따른 자동 갱신이 자기 시점에 정확히 반영됨.

### 3.3 만료는 저장하지 않음 (Lazy Evaluation)

```python
def active_batches(self, resource_id, current_step):
    return [b for b in self.holdings.get(resource_id, [])
            if b.expires_at > current_step]
```

만료된 배치를 DELETE 하지 않음. 읽을 때 `expires_at > current_step`로 필터링. 이렇게 하면:
- 만료 시점에 아무 처리도 안 함 (write 0회)
- step이 지나면 자동으로 "보이지 않게" 됨
- 정리는 세션 종료 시 배치로 한 번에

→ Redis EXPIRE의 lazy expiration 패턴과 동일.

### 3.4 wake_schedule 사전 등록

자원 만료 직전, 작업 완료 직후 같은 "이 시점에 뭔가 일어나야 함"은 미리 sorted set(Redis ZSET 등)에 등록:

```
wake_schedule (sorted set, score = step)
  step 5  → {actor_A}      ← actor A의 자원이 step 6에 만료, step 5에 깨워서 처리
  step 8  → {actor_B}      ← actor B의 작업이 step 8에 완료
  step 10 → {actor_A, C}   ← 주기 wake (예: 5 step마다)
```

중앙 스케줄러는 `get_wake_list(step)`로 누구를 깨울지 정함. 등록은 **action을 일으킨 worker가 직접** 함. 별도 시간 갱신 트리거가 필요 없음.

---

## 4. 해결 매핑: 충돌 vs Lazy Activation

| 문제 | 해결 |
|------|------|
| Race condition (액터 행동 vs 자동 갱신) | Worker가 액터를 단독 소유. 자동 갱신도 같은 worker에서 실행 → 직렬화됨 |
| Lost update | catch-up이 액터 행동 직전에 동기적으로 실행되므로 덮어쓰기 불가능 |
| Phantom read (step 전환 도중 조회) | 작업 loop마다 step 체크 + tick으로 즉시 catch-up |
| 휴면 액터의 처리 비용 | Lazy Activation으로 0 (시간만 흐름, 상태는 동결) |
| 만료 데이터 정리 부담 | Lazy evaluation (`expires_at > current_step`)로 write 0회 |
| 글로벌 락 필요성 | 액터 단위로 자연스럽게 격리되므로 락 불필요 |

핵심은 **"시간과 상태를 디커플링"** 하는 것. 시간은 글로벌 클럭이 흐르지만, 상태 변화는 액터가 깨어나는 시점에만 일어난다.

---

## 5. 레퍼런스 — 같은 패턴을 쓰는 시스템

### Microsoft Orleans (Halo, Xbox Live)

Virtual Actor 모델. Actor가 메시지를 받으면 활성화, 일정 시간 idle이면 자동 비활성화. 비활성 상태에서는 메모리/CPU 점유 0. 다음 메시지가 오면 마지막 상태에서 활성화.

> "An actor is always available conceptually, but only consumes resources when handling a message."

매핑: 액터 = Actor. wake = 활성화. 작업 종료 = 비활성화. catch-up = 마지막 상태에서 현재 시점까지 fast-forward.

### EVE Online (CCP Games)

수십만 명이 동시 접속하는 우주 게임. Solar System(맵의 한 영역)에 플레이어가 한 명도 없으면 그 시스템은 **틱이 돌지 않음**. NPC, 자원 spawn, 채굴 노드 회복 — 전부 멈춤. 플레이어가 진입하면 "마지막 누군가 있었던 시각"부터 현재까지의 변화를 한 번에 계산.

GDC 2015 발표 "Scalable Server Architecture"에서 공개. 이게 없으면 EVE는 운영 자체가 불가능.

### 농경 게임 (Stardew Valley, Harvest Moon)

작물 성장은 "심은 시각 + 성장 시간"으로 저장. 매 프레임 작물 카운터를 갱신하지 않음. 플레이어가 그 밭에 다가가서 화면에 표시될 때, "현재 시각 - 심은 시각"으로 성장 단계를 계산.

### Redis EXPIRE

Redis의 키 만료. **만료 시점에 키를 삭제하지 않음.** 만료 시각만 저장해두고, 다음 두 경우에만 정리:
1. **Lazy expiration**: 그 키를 GET 할 때 만료됐는지 확인 → 만료됐으면 그 시점에 삭제하고 nil 반환
2. **Active expiration**: 백그라운드에서 무작위 샘플링으로 일부만 정리

이게 시뮬레이션의 만료 처리와 거의 같은 패턴. 만료 시점에 아무것도 안 함, 읽을 때만 판단, 정리는 백그라운드.

### Akka (Lightbend)

Actor 모델 프레임워크. Akka Cluster Sharding은 Actor를 자동으로 클러스터에 분산하고, idle Actor를 passivate(휴면)시켰다가 메시지가 오면 다시 활성화. Orleans와 비슷.

---

## 6. 한계와 트레이드오프

### 한계 1: 휴면 액터의 "있어야 할 상태"는 외부에서 안 보임

다른 액터가 "actor A의 상태를 보고 싶다"고 하면? Actor A가 휴면 중이면 상태는 catch-up 안 된 stale 상태. 정확한 잔량을 알려면 calc-on-read가 필요:

```python
def get_state_view(state, clock, current_step):
    # 임시로 catch-up 시뮬레이션 (mutate 없이)
    return simulate_catch_up(state, clock, current_step)
```

또는 **외부 조회는 stale 데이터로 충분하다고 받아들이기** (일관성 vs 성능 트레이드오프).

### 한계 2: catch-up이 길어질 수 있음

액터가 100 step 자고 깨어나면 100 step분 catch-up이 한 번에 실행됨. 보통은 무시 가능하지만, step이 매우 많거나 자동 생성 종류가 많으면 wake latency가 튐.

대응: 휴면 시간에 상한을 두기 (예: 10 step 이상 안 깨어나면 강제 wake) — 보통 wake_schedule 주기 등록으로 자동 처리.

### 한계 3: catch-up 도중 에러 → 부분 적용

`activate_actor` 안에서 일부는 성공, 일부는 실패 → state가 부분만 갱신됨. 트랜잭션으로 묶거나 idempotent하게 만들어야 함.

대응: catch-up 전체를 PostgreSQL 트랜잭션으로 감싸고, 실패 시 rollback. `last_wake_step` 갱신도 같은 트랜잭션 안.

---

## 7. 핵심 정리

| 원칙 | 적용 |
|------|------|
| 시간과 상태를 디커플링 | 시간은 글로벌 클럭, 상태는 wake 시 처리 |
| Pull > Push | 만료/완료를 push하지 않음, 읽을 때 또는 wake 때 pull |
| Single-owner per actor | Worker가 액터를 독점 → 락 없이 직렬화 |
| Lazy evaluation | 만료 데이터는 저장하지 않고 read-time 필터 |
| 사전 등록 | 미래에 일어날 일은 wake_schedule에 미리 등록 |

이 5개 원칙이 동시에 작동해야 함. 하나라도 빠지면 다른 곳에서 락이나 폴링이 필요해짐.

---

## 8. References

- [Microsoft Orleans — Virtual Actor Pattern](https://learn.microsoft.com/en-us/dotnet/orleans/overview)
- [EVE Online: GDC 2015 — Scalable Server Architecture](https://www.gdcvault.com/play/1022186/EVE-Online-How-CCP)
- [Akka Cluster Sharding & Passivation](https://doc.akka.io/docs/akka/current/typed/cluster-sharding.html)
- [Redis EXPIRE — Lazy + Active expiration](https://redis.io/commands/expire/)
- Martin Kleppmann — *Designing Data-Intensive Applications*, Ch.5 Replication
- 관련 노트: `db-time-patterns.md` (저장 측면 deadline-based 패턴)
