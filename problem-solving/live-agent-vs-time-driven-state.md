# 살아있는 에이전트와 시간 기반 상태 갱신의 충돌

| 항목 | 내용 |
|------|------|
| Date | 2026-04-10 |
| Context | 에이전트가 자신의 상태를 자유롭게 읽고 쓰는 동안, 시간(턴)에 따라 자동으로 변해야 하는 것들(생산 완료, 자원 만료, 자동 생산)이 같은 데이터를 건드림 |
| 핵심 질문 | 두 갱신 주체가 같은 ERP/Inventory를 동시에 만질 때, 어떻게 일관성과 성능을 모두 잡는가? |
| 관련 | `db-time-patterns.md` (저장 측면), `turn-system-design.md` (구현) |

---

## 1. 문제 정의

AgentPit에서 ERP 한 행을 누가 건드리는지 살펴보면:

```
[Agent 자신의 행동]                      [시간이 지나면 자동으로]
- send_offer → 재고 잠금 검증            - 생산 완료 → 산출물 inventory 추가
- accept_offer → cash & inventory 변경   - 자원 만료 → 배치 spoilage 제거
- start_production → 입력 자원 소비      - 자동 생산 (Farm) → wheat/milk 추가
- sell_to_npc → 재고 차감
```

**둘 다 같은 ERP.inventory를 건드린다.** 두 주체가 동시에 접근하면:

- **Race condition**: agent가 wheat 5개로 거래 검증하는 사이, 만료 처리가 wheat 3개를 spoilage로 빼가면 거래는 실패해야 하나 성공하나?
- **Lost update**: 생산 완료가 inventory에 flour를 추가하는 동안, agent가 inventory를 읽고 쓰면 추가분이 덮어쓰여 사라짐
- **Phantom read**: agent가 turn 5에서 inventory 조회 후 의사결정하는데, 그 사이 turn 6으로 넘어가면서 milk가 만료됨 → 잘못된 결정

---

## 2. 후보 해결책들

### 2.1 Global Ticker (가장 단순, 가장 나쁨)

매 라운드마다 GM이 모든 에이전트의 상태를 일괄 갱신.

```python
def round_tick(round_number):
    for agent in all_agents:
        complete_production(agent)
        expire_batches(agent)
        run_auto_production(agent)
    # 그 다음에 에이전트 행동 처리
```

- ❌ 10만 에이전트 × 매 라운드 = 처리 폭발
- ❌ 잠든 에이전트도 처리 비용 발생
- ❌ 라운드 길이가 길어지거나 에이전트 수가 늘면 GM이 병목
- ❌ Tick 도중엔 에이전트 행동을 막아야 함 (글로벌 락)

### 2.2 Lock + Global Ticker

위의 변형. tick 도중 에이전트 락을 걸고, 행동은 락 해제 후.

- ❌ 락 경합으로 throughput 폭락
- ❌ 분산 환경에서 분산 락 필요 → 복잡도 폭발
- ❌ 본질적 비효율(잠든 에이전트 처리)은 그대로

### 2.3 Event-Driven Timer

생산 완료 시점, 만료 시점마다 콜백 등록 → 그 시점에 trigger.

```python
schedule.at(completion_round, lambda: complete_production(agent))
schedule.at(expiry_round, lambda: expire_batch(agent, batch))
```

- ✅ 잠든 에이전트는 콜백만 등록되어 있음 (가벼움)
- ❌ 콜백이 다른 콜백과 충돌 가능 (생산 완료 콜백 vs agent 행동)
- ❌ 콜백 수가 많아지면 timer wheel 관리 부담
- ❌ 콜백 실행 시점에 agent 행동 중이면 어떻게? 결국 락 필요

### 2.4 Lazy Activation (Orleans Virtual Actor) ← AgentPit 채택

**시간은 흐르되, 상태 갱신은 에이전트가 깨어날 때만 일어난다.**

```python
def activate_agent(erp, clock, role_id, orders, current_round):
    # 1) 잠든 동안 완료된 생산 처리
    completed = complete_production(erp.inventory, erp.erp_id, orders, current_round)
    
    # 2) 잠든 동안의 자동 생산 catch-up (last_wake_round+1 ~ current_round)
    process_auto_production(erp.inventory, role_id, current_round, clock.last_wake_round)
    
    # 3) clock 갱신
    clock.last_wake_round = current_round
    return completed
```

핵심 원리:
- **잠든 에이전트의 상태는 "동결"** — 누구도 안 건드림
- **깨어날 때 한 번에 catch-up** — `last_wake_round`부터 `current_round`까지 일괄 처리
- **catch-up은 같은 worker가 함** — agent 행동과 같은 컨텍스트에서 실행되므로 동시성 문제 없음
- **만료(spoilage)는 저장하지 않고 lazy 평가** — `expires_at > current_round` 필터로 읽을 때만 판단

---

## 3. AgentPit 구현 디테일

### 3.1 Worker가 단독 소유

```
[Celery Task]
   │
   ├── Agent A를 처리하는 Worker는 Agent A의 상태를 단독 소유
   │
   ├── 같은 라운드에 Agent A를 처리하는 Worker는 1개만 존재 (배타성)
   │
   └── 다른 Worker가 Agent A의 ERP를 동시에 건드릴 일이 없음
```

→ **에이전트 단위 직렬화**. 분산 락 없이 단일 worker context로 동시성 문제를 우회.

### 3.2 두 단계 catch-up: activate vs tick

```python
# wake 시점 1회 — 잠들었다 깨어남
activate_agent(erp, clock, role_id, orders, current_round)

# tool-use loop 도중 라운드 전진 시
tick_agent(erp, clock, role_id, orders, current_round)
```

`tick_agent`가 필요한 이유: tool-use loop이 길어서 (LLM 응답 대기, 여러 tool 호출) 진행 중에 GM의 라운드 타이머가 넘어갈 수 있음. 그러면 다음 tool 호출 직전에 라운드가 바뀐 걸 감지하고 1턴분 catch-up:

```python
# Worker의 tool-use loop
while not done:
    if clock.last_wake_round < gm.current_round:
        tick_agent(erp, clock, role_id, orders, gm.current_round)
    
    tool_call = llm.next_tool()
    result = execute_tool(tool_call)
    llm.feed(result)
```

이렇게 하면 agent가 자기 행동을 하는 도중에도, 라운드 전환에 따른 자동 생산/생산 완료가 자기 시점에 정확히 반영됨.

### 3.3 Spoilage는 저장하지 않음 (Lazy Evaluation)

```python
def active_batches(self, resource_id, current_round):
    return [b for b in self.holdings.get(resource_id, [])
            if b.expires_at > current_round]
```

만료된 배치를 DELETE 하지 않음. 읽을 때 `expires_at > current_round`로 필터링. 이렇게 하면:
- 만료 시점에 아무 처리도 안 함 (write 0회)
- 라운드가 지나면 자동으로 "보이지 않게" 됨
- 정리는 세션 종료 시 배치로 한 번에

→ `db-time-patterns.md`의 "Deadline-based 패턴" 그대로.

### 3.4 wake_schedule 사전 등록

자원 만료 직전, 생산 완료 직후 같은 "이 시점에 뭔가 일어나야 함"은 미리 Redis sorted set에 등록:

```
wake_schedule (Redis ZSET)
  R5  → {agent_A}      ← agent A의 milk가 R6에 만료, R5에 깨워서 처리하라고
  R8  → {agent_B}      ← agent B의 production이 R8에 완료
  R10 → {agent_A, C}   ← 주기 wake (e.g. 5라운드마다)
```

GM은 `get_wake_list(round)`로 누구를 깨울지 정함. 등록은 **action을 일으킨 worker가 직접** 함. 별도 시간 갱신 트리거가 필요 없음.

---

## 4. 해결 매핑: 충돌 vs 우리 접근

| 문제 | 해결 |
|------|------|
| Race condition (agent 행동 vs 자동 갱신) | Worker가 agent를 단독 소유. 자동 갱신도 같은 worker에서 실행 → 직렬화됨 |
| Lost update | catch-up이 agent 행동 직전에 동기적으로 실행되므로 덮어쓰기 불가능 |
| Phantom read (라운드 전환 도중 조회) | tool-use loop마다 라운드 체크 + tick_agent로 즉시 catch-up |
| 잠든 에이전트의 처리 비용 | Lazy Activation으로 0 (시간만 흐름, 상태는 동결) |
| 만료 데이터 정리 부담 | Lazy evaluation (`expires_at > current_round`)로 write 0회 |
| 글로벌 락 필요성 | 에이전트 단위로 자연스럽게 격리되므로 락 불필요 |

핵심은 **"시간과 상태를 디커플링"** 하는 것. 시간은 GM의 클럭이 흐르지만, 상태 변화는 에이전트가 깨어나는 시점에만 일어난다.

---

## 5. 레퍼런스 — 같은 패턴을 쓰는 시스템

### Microsoft Orleans (Halo, Xbox Live)

Virtual Actor 모델. Actor가 메시지를 받으면 활성화, 일정 시간 idle이면 자동 비활성화. 비활성 상태에서는 메모리/CPU 점유 0. 다음 메시지가 오면 마지막 상태에서 활성화.

> "An actor is always available conceptually, but only consumes resources when handling a message."

AgentPit 매핑: 에이전트 = Actor. wake = 활성화. tool-use loop 종료 = 비활성화. catch-up = 마지막 상태에서 현재 라운드까지 fast-forward.

### EVE Online (CCP Games)

수십만 명이 동시 접속하는 우주 게임. Solar System(맵의 한 영역)에 플레이어가 한 명도 없으면 그 시스템은 **틱이 돌지 않음**. NPC, 자원 spawn, 채굴 노드 회복 — 전부 멈춤. 플레이어가 진입하면 "마지막 누군가 있었던 시각"부터 현재까지의 변화를 한 번에 계산.

GDC 2015 발표 "Scalable Server Architecture"에서 공개. 이게 없으면 EVE는 운영 자체가 불가능.

### 농경 게임 (Stardew Valley, Harvest Moon)

작물 성장은 "심은 시각 + 성장 시간"으로 저장. 매 프레임 작물 카운터를 갱신하지 않음. 플레이어가 그 밭에 다가가서 화면에 표시될 때, "현재 시각 - 심은 시각"으로 성장 단계를 계산.

### Redis EXPIRE

Redis의 키 만료. **만료 시점에 키를 삭제하지 않음.** 만료 시각만 저장해두고, 다음 두 경우에만 정리:
1. **Lazy expiration**: 그 키를 GET 할 때 만료됐는지 확인 → 만료됐으면 그 시점에 삭제하고 nil 반환
2. **Active expiration**: 백그라운드에서 무작위 샘플링으로 일부만 정리

이게 우리 spoilage 처리와 거의 같은 패턴. 만료 시점에 아무것도 안 함, 읽을 때만 판단, 정리는 백그라운드.

### Akka (Lightbend)

Actor 모델 프레임워크. Akka Cluster Sharding은 Actor를 자동으로 클러스터에 분산하고, idle Actor를 passivate(휴면)시켰다가 메시지가 오면 다시 활성화. Orleans와 비슷.

---

## 6. 한계와 트레이드오프

### 한계 1: 잠든 에이전트의 "있어야 할 상태"는 외부에서 안 보임

다른 에이전트가 "agent A의 inventory를 보고싶다"고 하면? Agent A가 잠들어 있으면 inventory는 catch-up 안 된 stale 상태. 정확한 잔량을 알려면 calc-on-read가 필요:

```python
def get_inventory_view(erp, clock, role_id, current_round):
    # 임시로 catch-up 시뮬레이션 (mutate 없이)
    return simulate_catch_up(erp, clock, role_id, current_round)
```

또는 **외부 조회는 stale 데이터로 충분하다고 받아들이기** (일관성 vs 성능 트레이드오프).

AgentPit 결정: 에이전트 간 inventory 조회는 send_offer/accept_offer 시 검증 단계에서 양쪽 다 깨어있는 상태로 처리. 외부 stale 조회는 제공하지 않음.

### 한계 2: catch-up이 길어질 수 있음

에이전트가 100라운드 자고 깨어나면 100라운드분 catch-up이 한 번에 실행됨. 자동 생산이 라운드당 6 wheat 추가라면 600개 처리. 보통은 무시 가능하지만, 라운드가 매우 많거나 자동 생산 종류가 많으면 wake latency가 튐.

대응: 잠자는 시간에 상한 (e.g. 10라운드 이상 안 깨어나면 강제 wake) — 우리는 wake_schedule 주기 등록으로 자동 처리.

### 한계 3: catch-up 도중 에러 → 부분 적용

`activate_agent` 안에서 `complete_production` 성공, `process_auto_production`에서 실패 → ERP가 부분만 갱신됨. 트랜잭션으로 묶거나 idempotent하게 만들어야 함.

대응: catch-up 전체를 PostgreSQL 트랜잭션으로 감싸고, 실패 시 rollback. last_wake_round 갱신도 같은 트랜잭션 안.

---

## 7. 핵심 정리

| 원칙 | 적용 |
|------|------|
| 시간과 상태를 디커플링 | 시간은 GM 클럭, 상태는 agent wake 시 처리 |
| Pull > Push | 만료/완료를 push하지 않음, 읽을 때 또는 wake 때 pull |
| Single-owner per agent | Worker가 agent를 독점 → 락 없이 직렬화 |
| Lazy evaluation | 만료 데이터는 저장하지 않고 read-time 필터 |
| 사전 등록 | 미래에 일어날 일은 wake_schedule에 미리 등록 |

이 5개 원칙이 동시에 작동해야 함. 하나라도 빠지면 다른 곳에서 락이나 폴링이 필요해짐.

---

## 8. 참고

- Microsoft Orleans 공식 문서: Virtual Actor pattern
- EVE Online: GDC 2015 "Scalable Server Architecture"
- Akka: Cluster Sharding & Passivation
- Redis: 공식 문서 EXPIRE — Lazy + Active expiration
- 도서: "Designing Data-Intensive Applications" (Kleppmann), Ch.5 Replication
- AgentPit 내부: `dev-notes/problem-solving/db-time-patterns.md`, `docs/design/turn-system-design.md`
