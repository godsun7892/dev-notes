# Tool-use 런타임 레이어링 — 원리와 구조

LLM 에이전트가 tool을 호출하고, 그 tool이 검증·상태 변경·영속화·응답 포맷팅까지 수행하는 시스템을 **레이어로 어떻게 쪼개야 하는가**. 이 문서는 원리·설계 근거·예시 코드를 담는다. AgentPit 현재 상태는 프로젝트 문서(`docs/design/runtime-layer-responsibility.md`)에 있다.

## 한 줄 요약

tool-use 런타임은 7가지 서로 다른 성질의 관심사를 다루는데, 이들을 한 덩어리로 섞으면 유지비가 제곱으로 증가한다. **"아는 것"과 "모르는 것"의 경계**로 레이어를 나누고, 의존성 방향을 단일하게(shared → game → adapter → operation → tool adapter → runtime) 유지해야 한다.

---

## 1. 문제 — 섞기 쉬운 7개 관심사

tool-use 기반 에이전트 시스템이 매 호출마다 처리해야 하는 것:

| # | 관심사 | 성질 | 섞이면 생기는 문제 |
|---|---|---|---|
| 1 | LLM JSON Schema 인터페이스 | 런타임 컨텍스트 바인딩 | 매번 재생성 비용, agent_id 위조 |
| 2 | 도메인 액션 실행 (검증 + 상태 변경 + 이벤트) | 순수 비즈니스 로직 | 재사용 불가, 테스트 어려움 |
| 3 | 도메인 객체 표현 | 불변/순수 | 데이터 타입에 인프라 지식 누수 |
| 4 | 영속화 (DB/Redis/Mongo) | I/O, 인프라-specific | 도메인이 dialect에 묶임 |
| 5 | 인프라 추상 (Protocol) | interface | 교체 불가, 테스트 단위 분리 실패 |
| 6 | LLM 보안 (위조 방지) | 실행 컨텍스트 | agent가 다른 agent로 위장 가능 |
| 7 | LLM-bound 응답 포맷 | LLM과의 계약 | 비-LLM 호출자가 dict 파싱 강요받음 |

각 관심사의 **변경 빈도가 다르다**:
- 프롬프트 수정 (매주)
- tool 시그니처 (월 단위)
- 도메인 엔진 로직 (분기 단위)
- 인프라 어댑터 (반년 단위)
- shared 도메인 모델 (거의 불변)

자주 바뀌는 것을 거의 안 바뀌는 것에 섞으면, 안 바뀌는 것도 같이 흔들린다. 레이어 분리의 **1차적 근거는 변경 빈도**.

## 2. 이론적 기반

### 2.1 Functional Core / Imperative Shell
> Gary Bernhardt의 Boundaries talk(2012)에서 온 구분.

- **Functional Core**: 순수 함수로 구성된 비즈니스 로직. 입력 → 출력, 부작용 없음, 시간/IO 없음.
- **Imperative Shell**: IO, DB, 네트워크, 시계. Core를 감싼다.

AgentPit에서:
- Core: `libs/economy_sim/engines/*` (trade, production, npc, auto_prod)
- Shell: `apps/worker/*`, `apps/gamemaster/*`

엔진이 `datetime.now()` / `db.execute()` / `llm.call()`을 안 쓰는 게 Functional Core 규약의 표현.

### 2.2 Dependency Inversion Principle (DIP)
> 구체가 추상에 의존해야 한다. 추상이 구체에 의존하면 안 된다.

tool 함수(L6)는 `ChannelRepository(Protocol)`에 의존하지, `PgChannelRepository`에 의존하지 않는다. 그래서:
- 테스트 시 InMemory 구현 주입 가능 — **현재 AgentPit은 ADR-010에 따라 InMemory 어댑터 폐지, 통합 테스트만 운영**
- 인프라 교체 시 tool 함수 불변

### 2.3 "아는 것"과 "모르는 것"의 경계
레이어를 나누는 조작적 정의:

```
이 모듈이 import하는 것은 무엇인가?
이 모듈을 import하는 것은 무엇인가?
양쪽이 교차하면 경계 위반.
```

예: `apps/worker/adapters/postgres/channel_repo.py`가 `economy_sim.exceptions`를 import한다면 "게임-독립 어댑터가 게임 도메인을 알아야" 하는 것 — 경계 위반.

---

## 3. 레이어 모델

### 3.1 8층 구조

```
 ┌─────────────────────────────────────────────────────────┐
 │ L8. Runtime Entry                                       │
 │    Redis Streams loop, pool lifecycle, Runner.run, retry│
 └─────────────────────────────────────────────────────────┘
                           │ orchestrates
                           ▼
 ┌─────────────────────────────────────────────────────────┐
 │ L7. Tool Adapter  (Agents SDK 의존 격리)                │
 │    @function_tool closure, docstring, schema gen        │
 └─────────────────────────────────────────────────────────┘
                           │ delegates to
                           ▼
 ┌─────────────────────────────────────────────────────────┐
 │ L6. Domain Operation  (core tool function)              │
 │    validate + apply + emit event                        │
 └──────────────┬────────────────────┬─────────────────────┘
                │                    │
                ▼                    ▼
 ┌──────────────────────┐   ┌─────────────────────────────┐
 │ L5. Infra Adapter    │   │ L4. Game Domain             │
 │    PgXxxRepo         │   │    engines + game errors    │
 └──────────┬───────────┘   └──────────────┬──────────────┘
            │ implements                    │ imports
            ▼                              ▼
 ┌─────────────────────────────────────────────────────────┐
 │ L1–L3. shared                                           │
 │  · Pure Domain (Channel, Message, Agent)                │
 │  · Infra Protocol (XxxRepository, RepositoryError)      │
 │  · LLM Contract (tool_response helper, BaseErrorCode)   │
 └─────────────────────────────────────────────────────────┘
```

의존 방향은 **위에서 아래로만**. 아래 레이어가 위 레이어를 import하면 경계 위반.

### 3.2 레이어별 역할 상세

#### L1. Pure Domain (shared)
- 아는 것: 자기 필드, equality, 직렬화
- 모르는 것: DB 존재, LLM 존재, 게임 규칙
- 구성: `frozen=True` dataclass + `tuple` 컬렉션 + 방어적 복사
- 테스트: 단위 테스트 (의존성 zero)

#### L2. Infra Protocol (shared)
- 아는 것: "무엇을 한다" (read/write/find semantics)
- 모르는 것: SQL dialect, Redis 명령, 파일 포맷
- 구성: `typing.Protocol` + `@runtime_checkable`
- 테스트: 없음 (interface만)

#### L3. LLM-bound Contract (shared)
- 아는 것: tool 응답 dict 스펙 — `{status, code?, message?, ...data}`
- 모르는 것: 구체 code 값(도메인이 정의), 구체 domain 객체
- 구성: 얇은 빌더 함수 + 범용 ErrorCode 베이스
- 테스트: 단위 테스트

#### L4. Game Domain (economy_sim 등)
- 아는 것: 게임 규칙, 게임-specific 에러
- 모르는 것: DB, LLM, 사용자
- 구성: 순수 엔진 + 도메인 모델 + GameEvent factory + game-specific ErrorCode
- 테스트: 단위 테스트 (pure functions)

#### L5. Infra Adapter (app)
- 아는 것: 특정 인프라 dialect
- 모르는 것: 게임 규칙, LLM, 사용자
- 책임: Protocol 구현 + row↔domain 매핑 + 인프라 예외 → `RepositoryError`
- 테스트: 통합 테스트 (실 인프라)

#### L6. Domain Operation (app, "core tool function")
- 아는 것: Repository Protocol, 도메인 검증, 게임 규칙, 이벤트 발행
- 모르는 것: Agents SDK, JSON Schema, Runner
- 책임: **액션 실행의 source of truth**. `agent_id` / `round_number` 명시 인자로 받음
- 테스트: 통합 테스트 (Repository 주입)

#### L7. Tool Adapter (app)
- 아는 것: Agents SDK API, run-scope context
- 모르는 것: 도메인 로직 자체
- 책임: 얇은 closure wrap. LLM 위조 불가 인자 가림. docstring = 프롬프트
- 테스트: 단위 테스트 (docstring/schema 정합성)

#### L8. Runtime Entry
- 아는 것: consumer group, pool 수명, tracing
- 모르는 것: tool 내부
- 책임: 외부 세계 접점

---

## 4. 예시 코드 — AgentPit 스타일

### 4.1 L1 도메인 (`libs/shared/messaging.py`)

```python
from dataclasses import dataclass
from typing import Protocol, runtime_checkable


@dataclass(frozen=True)
class Channel:
    channel_id: str
    session_id: str
    channel_type: str
    member_agent_ids: tuple[str, ...]  # tuple (immutable)


@dataclass(frozen=True)
class Message:
    channel_id: str
    sender_id: str
    content: str
    round_number: int
    message_id: str


@runtime_checkable
class ChannelRepository(Protocol):
    async def create(self, channel: Channel) -> None: ...
    async def get(self, channel_id: str) -> Channel: ...  # 없으면 NotFoundError
    async def list_by_agent(self, agent_id: str, *, session_id: str) -> list[Channel]: ...
    # ↑ session_id는 required (cross-session leak 방어를 스키마 레벨로 강제)


@runtime_checkable
class MessageRepository(Protocol):
    async def create(self, message: Message) -> None: ...
    async def list_by_channel(
        self, channel_id: str, since_round: int | None = None,
    ) -> list[Message]: ...
```

### 4.2 L3 LLM-bound helper (`libs/shared/tool_response.py`)

```python
from enum import Enum
from typing import Any


class BaseErrorCode(str, Enum):
    # messaging 관련 (게임-독립)
    CHANNEL_NOT_FOUND = "channel_not_found"
    NOT_CHANNEL_MEMBER = "not_channel_member"
    # 범용
    INVALID_INPUT = "invalid_input"
    INTERNAL_ERROR = "internal_error"


def success(**data: Any) -> dict[str, Any]:
    return {"status": "success", **data}


def error(code: str, message: str, **extra: Any) -> dict[str, Any]:
    # code: Enum.value 또는 str. 여러 도메인의 ErrorCode를 받아들이기 위해 str로 받음
    return {"status": "error", "code": code, "message": message, **extra}
```

### 4.3 L5 Postgres Adapter

```python
from psycopg_pool import AsyncConnectionPool
from psycopg.rows import dict_row

from shared.messaging import Channel, ChannelRepository
from shared.repository import NotFoundError, RepositoryError


class PgChannelRepository:
    def __init__(self, pool: AsyncConnectionPool) -> None:
        self._pool = pool

    async def get(self, channel_id: str) -> Channel:
        try:
            async with self._pool.connection() as conn, conn.cursor(row_factory=dict_row) as cur:
                await cur.execute(
                    "SELECT channel_id, session_id, channel_type FROM channel WHERE channel_id = %s",
                    (channel_id,),
                )
                row = await cur.fetchone()
                if row is None:
                    raise NotFoundError(f"channel not found: {channel_id}")
                # ... member 로드 생략
                return self._row_to_domain(row, members)
        except (psycopg.OperationalError, psycopg.InterfaceError) as e:
            raise RepositoryError(f"channel.get failed: {e}") from e
```

**원칙**: `except Exception:` 금지. catch는 **특정 인프라 예외**만 잡아서 `RepositoryError` 체이닝.

### 4.4 L6 Domain Operation

```python
# apps/worker/tools/message_tools.py
import logging
from typing import Any

import uuid_utils

from shared.messaging import ChannelRepository, Message, MessageRepository
from shared.repository import NotFoundError
from shared.tool_response import BaseErrorCode, error, success

logger = logging.getLogger(__name__)


async def send_message(
    *,
    channel_id: str,
    content: str,
    sender_agent_id: str,       # LLM 위조 불가 인자 — tool adapter가 주입
    round_number: int,           # LLM 위조 불가 인자
    channel_repo: ChannelRepository,
    message_repo: MessageRepository,
) -> dict[str, Any]:
    # 1. 입력 검증
    if not content.strip():
        return error(BaseErrorCode.INVALID_INPUT.value, "Message content must not be empty.")

    # 2. 채널 존재 확인
    try:
        channel = await channel_repo.get(channel_id)
    except NotFoundError:
        logger.warning(
            "channel_not_found",
            extra={"channel_id": channel_id, "sender_agent_id": sender_agent_id},
        )
        return error(BaseErrorCode.CHANNEL_NOT_FOUND.value, "The channel no longer exists.")

    # 3. 멤버십 확인
    if sender_agent_id not in channel.member_agent_ids:
        logger.warning(
            "not_channel_member",
            extra={"channel_id": channel_id, "sender_agent_id": sender_agent_id},
        )
        return error(BaseErrorCode.NOT_CHANNEL_MEMBER.value, "You are not a member of this channel.")

    # 4. 부작용 실행
    message = Message(
        message_id=str(uuid_utils.uuid7()),
        channel_id=channel_id,
        sender_id=sender_agent_id,
        content=content,
        round_number=round_number,
    )
    await message_repo.create(message)

    logger.info(
        "message_sent",
        extra={
            "message_id": message.message_id,
            "channel_id": channel_id,
            "sender_agent_id": sender_agent_id,
            "round_number": round_number,
        },
    )
    return success(message_id=message.message_id)
```

**원칙**:
- 검증 → 조회 → 검증 → 부작용 → 로깅 → 반환 순서
- `logger.info("event_name", extra={...})` — f-string 금지 (구조화 로깅)
- `agent_id` / `round_number` 인자로 받음 — tool adapter 책임
- 반환은 **LLM-bound dict**. 게임 시스템이 직접 호출할 필요가 생기면 core 층 추가 분리 고려

### 4.5 L7 Tool Adapter (Agents SDK closure)

```python
# apps/worker/runtime/tools.py
from typing import Any

from agents import FunctionTool, function_tool

from shared.messaging import ChannelRepository, MessageRepository
from worker.tools.message_tools import read_channel, send_message


def make_message_tools(
    *,
    channel_repo: ChannelRepository,
    message_repo: MessageRepository,
    agent_id: str,               # closure로 바인딩 — LLM에 노출 X
    round_number: int,           # closure로 바인딩 — LLM에 노출 X
) -> list[FunctionTool]:
    """매 Runner.run() 호출마다 새로 생성. agent_id/round는 해당 run의 컨텍스트."""

    @function_tool
    async def send_message_tool(channel_id: str, content: str) -> dict[str, Any]:
        """Send a message to a channel you are a member of.

        Args:
            channel_id: Target channel identifier.
            content: Message body. Must be non-empty.
        """
        return await send_message(
            channel_id=channel_id,
            content=content,
            sender_agent_id=agent_id,
            round_number=round_number,
            channel_repo=channel_repo,
            message_repo=message_repo,
        )

    @function_tool
    async def read_channel_tool(
        channel_id: str, since_round: int | None = None,
    ) -> dict[str, Any]:
        """Read messages from a channel you are a member of.

        Args:
            channel_id: Target channel identifier.
            since_round: Return messages from this round onwards (inclusive).
        """
        return await read_channel(
            channel_id=channel_id,
            reader_agent_id=agent_id,
            since_round=since_round,
            channel_repo=channel_repo,
            message_repo=message_repo,
        )

    return [send_message_tool, read_channel_tool]
```

**closure 패턴의 이유**:
- `agent_id`가 tool 시그니처에 있으면 LLM이 아무 값이나 넣을 수 있음 → 다른 에이전트로 위장 가능
- `round_number`가 시그니처에 있으면 LLM이 과거/미래 라운드에 쓸 수 있음 → 타임라인 위조
- closure로 **run-scope 컨텍스트를 가두고**, LLM에는 "LLM이 조작해도 되는 인자"만 노출
- factory 함수 `make_message_tools`가 매 `Runner.run()` 호출마다 불림 — run마다 context가 다르므로

---

## 5. 흔한 안티패턴 5가지

### AP1. L6에서 SQL/Redis 직접 호출
```python
# 나쁜 예
async def send_message(..., conn: AsyncConnection) -> ...:
    await conn.execute("INSERT INTO message ...", ...)
```
Repository Protocol을 우회 → 테스트 불가, 인프라 교체 불가. **항상 Protocol 주입**.

### AP2. L7에서 검증 로직 작성
```python
# 나쁜 예
@function_tool
async def send_message_tool(channel_id: str, content: str) -> ...:
    if not content.strip():   # ← L6이 할 일
        return error(...)
    channel = await channel_repo.get(channel_id)  # ← L6이 할 일
    ...
```
tool adapter가 얇지 않으면 같은 로직을 다른 진입점(예: 테스트, 시스템 이벤트)에서 재사용 불가. **L7은 forward만**.

### AP3. L4/L6에서 `datetime.now()` 호출
```python
# 나쁜 예
message = Message(..., created_at=datetime.now())
```
시간은 shell이 주입해야 한다. DB `DEFAULT now()` 또는 L7에서 명시 주입. AgentPit CLAUDE.md 명시 제약.

### AP4. 게임-독립 모듈이 게임 패키지를 import
```python
# 나쁜 예
# apps/worker/adapters/postgres/channel_repo.py
from economy_sim.exceptions import NotFoundError  # ← 역방향 의존
```
어댑터는 여러 게임이 공유 가능해야 하는데 특정 게임 패키지를 알면 다른 게임이 import할 때 economy_sim 전부를 끌고 옴. **shared로 끌어올리기**.

### AP5. tool 함수가 LLM-bound dict와 도메인 객체를 **섞어서 반환**
```python
# 나쁜 예
async def send_message(...) -> Message | dict:
    if error:
        return {"status": "error", ...}
    return Message(...)
```
Union 반환은 호출자를 강요한다 — 매번 `isinstance` 체크. 한 타입으로 통일 (dict 또는 객체+raise).

---

## 6. Tool 응답 계약 설계

### 6.1 원칙: LLM은 구조화된 키-값 dict를 읽는 것이 가장 안정적
자유 문장(`return "Success! Message id = abc123"`)은 LLM이 파싱 오류를 일으킬 수 있다. **항상 dict**, 구조 고정.

### 6.2 공통 스키마
```python
# 성공
{"status": "success", "message_id": "01...", ...}

# 실패
{"status": "error", "code": "channel_not_found", "message": "The channel no longer exists."}
```

### 6.3 code는 enum-backed 문자열
- enum으로 정의 (오타 방지)
- LLM 응답에는 `.value`를 보내 JSON 직렬화 문제 피함
- **프롬프트에 가능한 code 목록을 포함** → LLM이 recovery 전략 짤 수 있음

### 6.4 message는 영어 권장, 고정 문자열
CLAUDE.md 에러 5분류 규약에서 "도메인 실패 → LLM"은 언어팩 미대상. 간결한 영어로 충분.

---

## 7. DI 인자 폭발 방지 — Context 객체 패턴

`send_message`가 현재 4개 인자 DI(`channel_repo`, `message_repo`, `sender_agent_id`, `round_number`). EventRepository, SessionRepository가 추가되면 인자가 6~8개로 불어남.

### 7.1 해법: frozen dataclass로 묶기

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ToolContext:
    channel_repo: ChannelRepository
    message_repo: MessageRepository
    event_repo: EventRepository  # 향후
    agent_id: str
    round_number: int


async def send_message(
    *, channel_id: str, content: str, ctx: ToolContext,
) -> dict[str, Any]:
    ...
    channel = await ctx.channel_repo.get(channel_id)
    ...
```

### 7.2 closure 바인딩 변경
```python
def make_message_tools(ctx: ToolContext) -> list[FunctionTool]:
    @function_tool
    async def send_message_tool(channel_id: str, content: str) -> ...:
        return await send_message(channel_id=channel_id, content=content, ctx=ctx)
    ...
```

### 7.3 도입 시점
- Repository 3개 이상 주입되기 시작할 때
- 현재는 2개라 YAGNI 적용 가능. **EventRepository 도입과 동시에 리팩터**가 비용 효율

---

## 8. 참고

- AgentPit 현 상태: `docs/design/runtime-layer-responsibility.md`
- 관련 원칙: `dev-notes/code-design/functional-core-engine-pattern.md` (L4 엔진 규약)
- 관련 원칙: `dev-notes/code-design/error-handling-and-logging-principles.md` (에러 5분류)
- 관련 원칙: `dev-notes/code-design/frozen-dataclass-dict-fields.md` (L1 구성 규칙)
- ADR: `docs/adr/010-repository-protocol.md` · `docs/adr/012-dispatch-runtime-redis-streams.md` · `docs/adr/013-messaging-shared-lib.md`
