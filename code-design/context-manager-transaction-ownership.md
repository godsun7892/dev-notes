# Context Manager 트랜잭션 오너십 — `async with` / `__aexit__` / ROLLBACK

| 항목 | 내용 |
|------|------|
| 핵심 질문 | `async with conn.transaction():` 블록에서 예외가 터지면 누가 언제 ROLLBACK을 호출하는가? 그리고 Repository 메서드가 외부 트랜잭션에 참여하려면 어떻게 구조를 바꿔야 하는가? |
| 관련 문서 | [exception-chaining.md](exception-chaining.md), [error-handling-and-logging-principles.md](error-handling-and-logging-principles.md), ADR-014 §D3 (conn 주입 A 패턴) |

---

## 1. `with`는 `try/finally`의 문법 설탕이다

이 등가 변환 하나만 머리에 있으면 나머지가 풀린다.

```python
with open("a.txt") as f:
    f.read()
```

Python 인터프리터가 이걸 내부적으로 이렇게 취급한다:

```python
cm = open("a.txt")
f = cm.__enter__()
try:
    f.read()
finally:
    cm.__exit__(exc_type, exc_val, exc_tb)   # 정상이든 예외든 반드시 호출
```

`async with`는 async 버전. `__enter__`/`__exit__` 자리에 `__aenter__`/`__aexit__`이 들어가고 앞에 `await`가 붙는다. 의미는 동일하다.

**불변 원칙**: `with` 블록을 빠져나가는 모든 경로 — 정상 종료, return, break, continue, raise — 에서 `__exit__`이 한 번 호출된다.

## 2. `__exit__`은 예외 정보를 인자로 받는다

`close()`만 하는 게 아니라 **블록 안에서 예외가 있었는지 보고 다르게 동작할 수 있다**.

```python
class MyContext:
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            print("정상 종료")
        else:
            print(f"예외로 종료: {exc_type.__name__}")
        # return True  → 예외 삼킴
        # return False/None → 예외 계속 전파
```

인터프리터가 블록을 빠져나올 때 **예외가 있었으면 그걸 `exc_type`에 담아** `__exit__`에 전달한다. 정상이면 `None, None, None`.

**반환값 규칙**:
- `True` 반환 → 예외 삼켜짐. 블록 밖으로 안 나감.
- `False`/`None`/반환 없음 → 예외 계속 전파.

## 3. psycopg `conn.transaction()`의 `__aexit__` 내부 구현

psycopg3의 `AsyncTransaction`을 의사코드로 풀면:

```python
class AsyncTransaction:
    async def __aenter__(self):
        await self._conn.execute("BEGIN")     # 트랜잭션 시작
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            await self._conn.commit()         # 정상 → COMMIT
        else:
            await self._conn.rollback()       # 예외 → ROLLBACK
        # return 없음 → 예외 계속 전파
```

그래서 **ROLLBACK을 내가 직접 부를 이유가 없다**. `async with conn.transaction():` 블록 안에서 예외를 raise만 하면, context manager가 알아서 ROLLBACK을 실행한다. 이게 "트랜잭션 오너십은 context manager에게 있다"의 의미.

## 4. 실패 시나리오 추적 — `channel_repo.create`

실제 프로젝트 코드 (`apps/worker/src/worker/adapters/postgres/channel_repo.py`):

```python
async def create(self, channel: Channel) -> None:
    sql_channel = "INSERT INTO messaging.channel (...) VALUES (...)"
    sql_member = "INSERT INTO messaging.channel_member (...) VALUES (...)"
    async with self._pool.connection() as conn:
        async with conn.transaction():
            await conn.execute(sql_channel, (...))
            for agent_id in channel.member_agent_ids:
                await conn.execute(sql_member, (...))
```

`for` 루프 안에서 두 번째 member INSERT가 `ForeignKeyViolation`으로 실패했다고 가정. 일어나는 순서:

1. `conn.execute(sql_member, ...)`가 `psycopg.errors.ForeignKeyViolation`을 raise.
2. 예외가 `for` 루프 밖으로 전파 → 안쪽 `async with conn.transaction()` 블록의 끝에 도달.
3. Python이 `txn.__aexit__(ForeignKeyViolation, ...)` 호출 → **`await self._conn.rollback()` 실행**. channel INSERT까지 같이 되돌아감.
4. `__aexit__`이 예외를 삼키지 않음 → 예외가 바깥 `async with self._pool.connection()` 블록의 끝에 도달.
5. Python이 `conn_cm.__aexit__(ForeignKeyViolation, ...)` 호출 → **conn을 pool에 반납**.
6. 예외가 `create` 함수 밖으로 전파 → 호출자가 받음.

**정리**:
- 안쪽 `with` (`conn.transaction()`): ROLLBACK 담당
- 바깥쪽 `with` (`pool.connection()`): conn 반납 담당
- 둘 다 예외를 막지 않음

정상 시나리오면 3단계가 `commit()`, 나머지는 같다.

## 5. Repository가 외부 트랜잭션에 참여하려면

### 문제

원래 ERP repo `save`는 이렇게 생겼었다:

```python
async def save(self, erp):
    async with self._pool.connection() as conn, conn.transaction():
        await self._upsert(conn, ...)
        ...
```

`execute_trade`처럼 **seller save + buyer save + offer 상태 전환 + trade INSERT**가 한 트랜잭션이어야 하는 시나리오에서, 이 `save`를 두 번 부르면 **두 개의 독립된 트랜잭션**이 생긴다. seller 커밋 후 buyer 실패 시 seller는 이미 COMMIT 상태. 원자성 깨짐.

### 해결 — connection 주입 (A 패턴)

ADR-014 §D3:

```python
async def save(self, erp, *, conn: Any = None):
    try:
        if conn is None:
            async with self._pool.connection() as c, c.transaction():
                await self._save(erp, c)           # Repo가 트랜잭션 소유
        else:
            await self._save(erp, conn)             # caller가 트랜잭션 소유
    except Exception as e:
        raise RepositoryError("save failed") from e

async def _save(self, erp, conn):
    # SQL만 실행. 트랜잭션 경계 모름.
    await conn.execute(sql_upsert_erp, ...)
    ...
```

두 호출 경로:

```python
# 경로 1 — 단독 save (기존 호출자 전부 이 경로, 행동 불변)
await erp_repo.save(erp)

# 경로 2 — multi-repo 원자 트랜잭션 (execute_trade 같은 use-case)
async with pool.connection() as conn, conn.transaction():
    await erp_repo.save(seller, conn=conn)
    await erp_repo.save(buyer, conn=conn)
    await offer_repo.mark_accepted(offer_id, conn=conn)
    await trade_repo.create(trade, conn=conn)
# 블록 끝에서 COMMIT. 중간 raise 시 4개 write 전체 ROLLBACK.
```

**두 경로 모두 ROLLBACK은 "가장 안쪽 `conn.transaction()` 블록의 `__aexit__`"이 담당한다.** 경로 1에선 save 내부의 `c.transaction()`, 경로 2에선 caller의 `conn.transaction()`. `_save`는 어느 경우에도 트랜잭션을 모른다.

## 6. 왜 wrap은 public 경계에서 한 번만

`_save`에도 try/except + `RepositoryError` wrap을 넣으면 **이중 wrap** 문제가 생긴다.

```python
async def _save(self, erp, conn):
    try:
        await conn.execute(...)
    except Exception as e:
        raise RepositoryError("inner") from e   # 1차 wrap
```

```python
async def save(self, erp, *, conn=None):
    try:
        await self._save(erp, c_or_conn)
    except Exception as e:
        raise RepositoryError("save failed") from e   # 2차 wrap
```

호출자가 받는 예외:

```
RepositoryError: save failed: RepositoryError('inner: UniqueViolation...')
```

두 번 감쌌다. `e.__cause__`로 파고들어야 원인이 나온다. 로그도 두 레이어에서 같은 사건을 찍는다([error-handling-and-logging-principles.md](error-handling-and-logging-principles.md)의 "같은 사건 한 번만 로그" 원칙 위반).

**규칙**: 예외 wrapping은 **Repo의 public 경계(`save`, `create`, `mark_*` 등)에서 한 번만**. 내부 헬퍼(`_save`)는 raw 예외를 통과시킨다.

## 7. Java `try-with-resources`와의 비교

Java에 비슷한 문법이 있다:

```java
try (Connection conn = pool.getConnection();
     Statement stmt = conn.createStatement()) {
    stmt.execute("...");
}
```

블록이 끝날 때 `AutoCloseable.close()`가 역순으로 호출된다. Python `with`와 거의 같은 동작.

| 항목 | Python `with` | Java `try-with-resources` |
|---|---|---|
| 정상 종료 시 cleanup | `__exit__(None, None, None)` | `close()` |
| 예외 종료 시 cleanup | `__exit__(예외 정보)` | `close()` (예외는 인자로 전달 안 됨) |
| 예외 삼키기 가능 | `__exit__`이 `True` 반환 | 불가능 (close()는 그냥 void) |
| 예외 정보 접근 | `__exit__` 인자로 받음 | 받지 않음 |

**차이**: Java `close()`는 예외 정보를 모른다. 그래서 Java에선 "트랜잭션 컨텍스트 매니저가 ROLLBACK/COMMIT을 자동 선택" 같은 패턴을 문법만으로 못 만든다. 보통 `try/catch/finally`로 명시적으로 쓴다:

```java
conn.setAutoCommit(false);
try {
    stmt.execute("...");
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
    throw e;
}
```

Python의 `conn.transaction()`은 **이 패턴을 context manager가 추상화**한다. 그래서 사용자 코드가 깨끗해진다.

## 8. 이 프로젝트 적용 사례

### Case A — 단일 Repo 다중 INSERT (`channel_repo.create`)

위 §4에서 본 대로. 한 트랜잭션에 channel + 여러 channel_member INSERT. Repo가 트랜잭션 소유(`conn=None` 경로만 필요한 케이스).

### Case B — Multi-repo 원자 트랜잭션 (`execute_trade`, 미구현)

위 §5에서 본 대로. caller(use-case 코드)가 트랜잭션 소유. `conn=conn` 주입.

### Case C — 기존 호출자 보존 (`read_my_erp` tool)

`read_my_erp` tool은 `erp_repo.get_by_agent(...)`만 부름. write 아니므로 conn 주입과 무관. `save`의 시그니처 변경(`*, conn=None` 추가)이 기존 호출자에 **영향 없는 이유**: 기본값이 `None` → 경로 1로 분기 → 예전 동작과 완전히 동일.

## 9. 자주 하는 실수

### 실수 1 — `__aexit__` 안에서 COMMIT/ROLLBACK을 직접 부름

```python
async def bad():
    async with pool.connection() as conn, conn.transaction():
        try:
            await conn.execute(...)
        except Exception:
            await conn.rollback()   # ❌ 중복
            raise
```

`conn.transaction()`의 `__aexit__`이 이미 ROLLBACK을 호출한다. 직접 호출하면 "이미 종료된 트랜잭션을 다시 ROLLBACK" 에러가 터지거나, 최소한 중복 작업.

**교정**:

```python
async def good():
    async with pool.connection() as conn, conn.transaction():
        await conn.execute(...)     # 실패하면 그냥 raise, 블록이 ROLLBACK 알아서 함
```

### 실수 2 — 외부 트랜잭션 안에서 새 트랜잭션 열기

```python
async def bad(pool):
    async with pool.connection() as conn, conn.transaction():
        await erp_repo.save(erp)   # ❌ save가 내부에서 또 BEGIN
```

`save`가 내부에서 자체 `conn.transaction()`을 열면, 외부 트랜잭션과 독립된 트랜잭션이 돼서 원자성 깨진다. ADR-014 §D3이 `conn=conn` 주입을 도입한 이유.

**교정**:

```python
async def good(pool):
    async with pool.connection() as conn, conn.transaction():
        await erp_repo.save(erp, conn=conn)   # ✅ caller 트랜잭션 참여
```

### 실수 3 — `_save`에서 try/except로 raw 예외 삼킴

```python
async def bad_save(self, erp, conn):
    try:
        await conn.execute(...)
    except Exception:
        logger.error("failed")    # ❌ 예외 삼킴
```

예외를 삼키면 `__aexit__`이 "정상 종료" 로 판단 → **COMMIT이 실행됨**. 데이터 손상 가능성. 로그만 찍고 raise하지 않으면 안 된다.

**교정**: `_save`는 try/except 없이 raw 예외 통과. 로깅은 public `save`의 wrap 지점에서.

## 10. 요약

| 키워드 | 한 줄 |
|--------|------|
| `with` = `try/finally` | 블록 종료 시 반드시 `__exit__` 호출. 정상/예외 모두. |
| `__aexit__(예외정보)` | 예외 유무를 인자로 받아 COMMIT/ROLLBACK 분기 |
| 트랜잭션 오너십 | `conn.transaction()` context manager가 담당. 사용자 코드는 raise만 하면 됨 |
| conn 주입 (A 패턴) | multi-repo 원자성 — caller가 트랜잭션 소유, Repo는 `conn=conn` 받아 참여 |
| wrap은 한 번만 | public 경계(`save`)에서 한 번. 내부 헬퍼(`_save`)는 raw 예외 통과 |

---

## References

- [PEP 343 — The "with" Statement](https://peps.python.org/pep-0343/) — `with` 문법과 `__enter__`/`__exit__` 도입
- [PEP 492 — Coroutines with async and await syntax](https://peps.python.org/pep-0492/) — `async with` 도입
- [Python docs — Context Manager Types](https://docs.python.org/3/library/stdtypes.html#context-manager-types) — `__enter__`/`__exit__` 사양
- [Python docs — contextlib](https://docs.python.org/3/library/contextlib.html) — `AsyncExitStack`, `asynccontextmanager`
- [psycopg3 — Transaction contexts](https://www.psycopg.org/psycopg3/docs/basic/transactions.html) — `conn.transaction()` 공식 문서
- [ADR-014 §D3](../../docs/adr/014-state-oriented-persistence-pivot.md) — conn 주입 A 패턴 의사결정
- [exception-chaining.md](exception-chaining.md) — `from e` / `from None` / 암묵적 체이닝
- [error-handling-and-logging-principles.md](error-handling-and-logging-principles.md) — "같은 사건 한 번만 로그" 원칙
