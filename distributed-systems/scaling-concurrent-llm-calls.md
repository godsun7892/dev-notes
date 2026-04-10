# 대규모 동시 LLM API 호출 스케일링

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 수만~수십만 클라이언트가 동시에 LLM API tool-use loop를 돌릴 때 어떻게 효율적으로 디스패칭하는가? |

---

## 1. 기본 문제

LLM API 호출은 본질적으로 **느리고(수 초) I/O bound**다. 한 요청이 1~5초 걸리고, 그 동안 CPU는 거의 idle. 따라서:

- **동시성이 높을수록 처리량 ↑**: 한 프로세스에서 수백 개 동시 호출 가능
- **CPU 기반 스케일링은 비효율**: CPU는 대부분 놀고 있음
- **rate limit 관리**: provider별로 제한이 있으므로 backoff/retry 필요

각 사용자가 자기 API key를 쓰는 경우 (multi-tenant), provider 측 rate limit은 사용자별로 독립적이므로 디스패칭 효율이 병목이 된다.

---

## 2. Tool-Use Loop는 프레임워크 없이 가능

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
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result,
            })
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
| 오버헤드 | ~14ms (벤치마크) | ~3ms |

순차적인 단순 tool-use loop면 Raw SDK 또는 OpenAI Agents SDK가 충분. LangGraph는 그래프 분기/체크포인트/병렬 실행이 필요할 때 가치가 있음.

---

## 3. 스케일링 아키텍처: 큐 + 워커 풀

```
요청 발생 → 내구성 큐 → 워커 풀 → LLM API
             (Redis)     (stateless,     (provider별 rate limit
                          수평 확장)       + connection pooling)
```

### Celery + Redis 패턴

```
스케줄러
  → Redis Sorted Set 또는 Stream에 task 등록
  → Celery Worker가 꺼내서 tool-use loop 실행
  → 결과는 다시 DB/큐에
```

### Celery 오버헤드

| 항목 | 소요 시간 |
|------|----------|
| 태스크 직렬화/역직렬화 | ~2ms |
| Redis 왕복 | ~1-2ms |
| **Celery 오버헤드 합계** | **~3-5ms** |
| LLM API 호출 | 1,000-5,000ms |

오버헤드 0.1% 미만. 체감 차이 없음. Celery를 쓰는 이유는 성능이 아니라 **수평 확장과 안정성**.

### 워커 동시 처리량

LLM API 호출은 I/O bound:

```
Celery prefork (기본, sync):
  --concurrency=10 → 동시 10개 → CPU 낭비

Celery gevent (async):
  --pool=gevent --concurrency=500 → 동시 500개 → 효율적
```

I/O bound이므로 gevent 또는 asyncio 기반 pool이 압도적 유리.

### 서버당 용량 (2코어, 4GB 기준)

```
워커 프로세스 1개 = ~100-150MB RAM
gevent pool, concurrency=500
→ 워커 2-3개 → 서버당 동시 1,000-1,500개
```

---

## 4. Connection Pooling

매 요청마다 새 TCP/TLS 연결을 만들면 핸드셰이크 비용이 큼 (수십~수백 ms).

```python
# ❌ Bad — 매 호출마다 새 connection
for prompt in prompts:
    client = OpenAI()  # 새 client 매번
    response = client.chat.completions.create(...)

# ✅ Good — client 재사용
client = OpenAI()  # 한 번만
for prompt in prompts:
    response = client.chat.completions.create(...)
```

OpenAI/Anthropic Python SDK는 내부적으로 `httpx.Client`를 재사용 (HTTP keep-alive). client 인스턴스를 재사용하면 자동으로 connection pooling.

### asyncio.Semaphore로 동시성 제한

```python
import asyncio

semaphore = asyncio.Semaphore(100)  # 동시 100개 제한

async def call_llm(prompt):
    async with semaphore:
        return await client.messages.create(...)

results = await asyncio.gather(*[call_llm(p) for p in prompts])
```

OS 파일 디스크립터 한계 (`ulimit -n`)를 넘으면 `Too many open files` 에러. semaphore로 미리 제한.

---

## 5. 오토스케일링

### K8s + KEDA (큐 깊이 기반)

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

## 6. Rate Limit 관리

LLM provider는 두 종류 limit이 있음:
- **RPM** (Requests Per Minute): 분당 요청 수
- **TPM** (Tokens Per Minute): 분당 토큰 수

### 클라이언트 측 처리

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(5),
       wait=wait_exponential(multiplier=1, min=2, max=60))
def call_with_retry():
    return client.messages.create(...)
```

`tenacity` 같은 라이브러리로 exponential backoff. provider가 `429 Too Many Requests` 반환 시 자동 재시도.

### 멀티 테넌트

각 사용자가 자기 API key를 쓰면 rate limit이 사용자별로 독립적이라 단일 limit로 묶이지 않음. 단, 사용자별 토큰 비용 추적 필요.

---

## 7. 검증된 레퍼런스

| 레퍼런스 | 내용 | 출처 |
|----------|------|------|
| LiteLLM Proxy | 사용자별 API key 라우팅, Redis 분산 rate limiting | github.com/BerriAI/litellm |
| Hookdeck/Outpost | 1000억+ 웹훅 처리, 큐 → 워커 → 전송 패턴 | hookdeck.com, github.com/hookdeck/outpost |
| aiohttp 벤치마크 | 단일 프로세스 초당 ~1,800 req, Semaphore(1000) 필수 | Pawel Miech 블로그 |
| LangGraph + Celery | Celery 태스크 안에서 LangGraph 실행, 50만 태스크/일 | zkhan.in 블로그 |
| LangGraph Platform | 인스턴스당 동시 10개, 최대 10 컨테이너/배포 | LangChain 공식 support docs |
| Temporal + LangGraph POC | Temporal Activity 안에 LangGraph, 크래시 복구 가능 | github.com/domainio/temporal-langgraph-poc |

---

## 8. 대규모 시스템 공통 패턴

모든 대규모 분산 LLM 시스템이 공유하는 구조:

```
요청 발생 → 내구성 큐 → 워커 풀 → API 호출
             (Redis/Kafka)  (stateless,     (connection pooling,
                             수평 확장)       동시성 제한)
```

핵심 원칙:
1. **큐 우선**: 요청을 큐에 먼저 넣고, 워커가 꺼내서 처리. 트래픽 스파이크 흡수
2. **동시성 제한**: `asyncio.Semaphore`로 동시 연결 수 제한 (OS FD 한계)
3. **Connection pooling**: HTTP 연결 재사용. 요청당 새 TCP 연결은 큰 비용
4. **Stateless 워커**: 워커에 상태 없음. 로드밸런서 뒤에서 자유롭게 추가/제거
5. **Backoff & retry**: provider rate limit에 대비
6. **큐 깊이 기반 스케일링**: CPU가 아닌 큐 길이로 워커 수 조절

---

## 9. References

- [Anthropic — Tool Use](https://platform.claude.com/docs/en/docs/agents-and-tools/tool-use/handle-tool-calls)
- [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)
- [LiteLLM (multi-provider proxy)](https://github.com/BerriAI/litellm)
- [Celery — gevent pool](https://docs.celeryq.dev/en/stable/userguide/concurrency/eventlet.html)
- [KEDA — Kubernetes Event-driven Autoscaling](https://keda.sh/)
- [tenacity — Retrying library for Python](https://tenacity.readthedocs.io/)
- [aiohttp — asyncio HTTP client/server](https://docs.aiohttp.org/)
- [Hookdeck — Webhook Infrastructure Guide](https://hookdeck.com/webhooks/guides/webhook-infrastructure-guide)
