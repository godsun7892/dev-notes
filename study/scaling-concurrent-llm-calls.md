# 대규모 동시 LLM API 호출 스케일링

## 핵심 문제

10만 에이전트가 각각 tool-use loop를 돌릴 때, 어떻게 동시 LLM API 호출을 효율적으로 처리하는가?

각 에이전트는 **다른 API key**를 사용하므로 제공자 측 rate limit은 문제가 아님. 우리 서버의 디스패칭 효율이 병목.

---

## 1. Tool-Use Loop는 프레임워크 없이 가능

Anthropic/OpenAI SDK만으로 ~20줄에 구현 가능:

```python
messages = [{"role": "user", "content": prompt}]
while True:
    response = client.messages.create(model=..., tools=tools, messages=messages)
    messages.append({"role": "assistant", "content": response.content})
    if response.stop_reason != "tool_use":
        break
    tool_results = []
    for block in response.content:
        if block.type == "tool_use":
            result = execute_tool(block.name, block.input)
            tool_results.append({"type": "tool_result", "tool_use_id": block.id, "content": result})
    messages.append({"role": "user", "content": tool_results})
```

출처: [Anthropic 공식 문서 - Handle Tool Calls](https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/handle-tool-calls)

### LangGraph vs Raw SDK

| | LangGraph | Raw SDK |
|---|---|---|
| Tool-use loop | 제공 | 직접 구현 (~20줄) |
| Checkpointing | 내장 | 직접 구현 |
| 복잡한 그래프 | 분기/병렬/서브그래프 | 불필요하면 과잉 |
| 스트리밍 | 토큰 단위 | 불필요하면 과잉 |
| 관찰성 | LangSmith 연동 | 직접 구현 |

AgentPit 에이전트는 순차 tool-use loop이므로 Raw SDK로 충분.

---

## 2. 스케일링 아키텍처: Raw SDK + Celery + Redis

```
wake_schedule (Redis)
  → 깨울 에이전트 목록 조회
  → Celery Task Queue (Redis broker)에 등록
  → Worker들이 꺼내서 tool-use loop 실행 (각자 다른 API key)
  → 응답 actions 실행 (trade, produce 등)
```

### Celery 오버헤드

| 항목 | 소요 시간 |
|------|----------|
| 태스크 직렬화/역직렬화 | ~2ms |
| Redis 왕복 | ~1-2ms |
| **Celery 오버헤드 합계** | **~3-5ms** |
| LLM API 호출 | 1,000-5,000ms |

오버헤드 0.1% 미만. 체감 차이 없음. Celery를 쓰는 이유는 성능이 아니라 수평 확장과 안정성.

### 워커 동시 처리량

LLM API 호출은 I/O bound (CPU 안 쓰고 네트워크 응답 대기):

```
Celery prefork (기본, sync):
  --concurrency=10 → 동시 10개 → CPU 낭비

Celery gevent (async):
  --pool=gevent --concurrency=500 → 동시 500개 → 효율적
```

I/O bound이므로 gevent pool이 압도적 유리.

### 서버당 용량 (2코어, 4GB 기준)

```
워커 프로세스 1개 = ~100-150MB RAM
gevent pool, concurrency=500
→ 워커 2-3개 → 서버당 동시 1,000-1,500개
→ 10만 동시 = 약 100대 서버 (최악의 경우)
```

Lazy Activation으로 실제 동시 깨어남은 훨씬 적음.

---

## 3. 오토스케일링

### K8s + KEDA (Redis 큐 깊이 기반)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  triggers:
  - type: redis
    metadata:
      listName: celery_task_queue
      listLength: "10"  # 큐에 10개 쌓이면 pod 추가
```

CPU 기반 스케일링은 I/O bound에 안 맞음. **큐 깊이 기반**이 정답.

### AWS ECS

```
CloudWatch 커스텀 메트릭 → Redis 큐 길이 퍼블리싱
→ Auto Scaling Policy: 큐 > 100이면 태스크 추가
→ EC2 Auto Scaling: 인스턴스 부족하면 EC2 추가
```

---

## 4. 검증된 레퍼런스

| 레퍼런스 | 내용 | 출처 |
|----------|------|------|
| LiteLLM Proxy | 사용자별 API key 라우팅, Redis 분산 rate limiting | github.com/BerriAI/litellm |
| Hookdeck/Outpost | 1000억+ 웹훅 처리, 큐 → 워커 → 전송 패턴 | hookdeck.com, github.com/hookdeck/outpost |
| aiohttp 벤치마크 | 단일 프로세스 초당 ~1,800 req, Semaphore(1000) 필수 | Pawel Miech 블로그 |
| LangGraph + Celery | Celery 태스크 안에서 LangGraph 실행, 50만 태스크/일 | zkhan.in 블로그 |
| LangGraph Platform | 인스턴스당 동시 10개, 최대 10 컨테이너/배포 | LangChain 공식 support docs |
| Temporal + LangGraph POC | Temporal Activity 안에 LangGraph, 크래시 복구 가능 | github.com/domainio/temporal-langgraph-poc |

---

## 5. 대규모 시스템 공통 패턴

모든 대규모 시스템 (Hookdeck, iFood, LangGraph Platform)이 공유하는 구조:

```
요청 발생 → 내구성 큐 → 워커 풀 → API 호출
             (Redis)     (stateless,     (connection pooling,
                          수평 확장)       동시성 제한)
```

핵심 원칙:
1. **큐 우선**: 요청을 큐에 먼저 넣고, 워커가 꺼내서 처리. 트래픽 스파이크 흡수.
2. **동시성 제한**: asyncio.Semaphore로 동시 연결 수 제한 (OS 파일 디스크립터 한계).
3. **Connection pooling**: HTTP 연결 재사용. 요청당 새 TCP 연결은 최대 성능 저하 요인.
4. **Stateless 워커**: 워커에 상태 없음. 로드밸런서 뒤에서 자유롭게 추가/제거.
