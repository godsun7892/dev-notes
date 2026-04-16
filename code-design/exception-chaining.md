# Exception Chaining — `from None` vs `from e` vs 암묵적

| 항목 | 내용 |
|------|------|
| 핵심 질문 | except 블록에서 새 예외를 raise할 때 원본 예외를 어떻게 처리하는가? |
| 관련 문서 | [error-handling-and-logging-principles.md](error-handling-and-logging-principles.md) §11 안티패턴 |

---

## 1. Python의 예외 체이닝 세 가지

### 암묵적 체이닝 (implicit chaining)

except 블록 안에서 새 예외를 raise하면 Python이 자동으로 원본을 `__context__`에 건다.

```python
try:
    d = {}
    d["missing"]
except KeyError:
    raise ValueError("bad input")
```

```
Traceback (most recent call last):
  File "...", line 3, in ...
KeyError: 'missing'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "...", line 5, in ...
ValueError: bad input
```

"During handling of the above exception" — 마치 에러 처리 도중 *또 다른 에러*가 터진 것처럼 보인다.
의도적 변환인데 사고처럼 읽힌다.

### 명시적 체이닝 (`from e`)

"이 예외의 직접적 원인은 e다"를 선언한다. `__cause__`에 원본이 담긴다.

```python
try:
    conn.execute(query)
except psycopg2.OperationalError as e:
    raise RepositoryError("db query failed") from e
```

```
Traceback (most recent call last):
  File "...", line 3, in ...
psycopg2.OperationalError: connection refused

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "...", line 5, in ...
RepositoryError: db query failed
```

"The above exception was the **direct cause**" — 인과 관계가 명확하다.

### 체이닝 끊기 (`from None`)

"이전 예외는 구현 디테일이니 보여주지 마라." `__cause__`를 None으로, `__suppress_context__`를 True로 설정.

```python
try:
    return self._store[key]
except KeyError:
    raise NotFoundError(f"entity {key!r} not found") from None
```

```
Traceback (most recent call last):
  File "...", line 5, in ...
NotFoundError: entity 'abc' not found
```

원본 `KeyError`가 트레이스백에 안 나온다.

---

## 2. 언제 어떤 걸 쓰는가

| 상황 | 체이닝 | 이유 |
|------|--------|------|
| 인프라 예외 → 도메인 예외 래핑 | `from e` | 원인이 인프라 장애임을 운영자가 알아야 함 |
| 내부 구현 디테일 → 공개 API 예외 변환 | `from None` | 원본은 호출자에게 노이즈 |
| except 안에서 진짜 다른 에러가 터짐 | 암묵적 (그대로) | Python 기본 동작이 맞음. 실제로 사고 |

### 판단 기준

> **"트레이스백을 읽는 사람에게 원본 예외가 유용한 정보인가?"**
>
> - Yes → `from e` (원인 보존)
> - No → `from None` (노이즈 제거)
> - 모르겠다 → 일단 `from e` (정보는 있는 게 없는 것보다 나음)

---

## 3. 이 프로젝트의 적용 사례

### Case A: Engine 내부 precondition → 도메인 실패 변환

```python
# production.py — start_production
for resource_id, qty in recipe.inputs.items():
    try:
        new_inv, _ = new_inv.with_consumed(resource_id, qty, current_round)
    except InsufficientConsumeError:
        available = inventory.available_quantity(resource_id, current_round)
        raise InsufficientStockError(resource_id, qty, available) from None
```

`InsufficientConsumeError`는 Inventory의 내부 precondition 위반 (Case 5).
`InsufficientStockError`는 호출자에게 전달되는 도메인 실패 (Case 1).
의도적 변환이므로 `from None`으로 원본을 숨긴다.

`from None`이 없으면:
```
economy_sim.exceptions.InsufficientConsumeError: wheat

During handling of the above exception, another exception occurred:

  ...
economy_sim.exceptions.InsufficientStockError
```

Worker/Tool 레이어가 `InsufficientStockError`만 보면 되는데,
`InsufficientConsumeError`가 같이 올라오면:
1. 로그에 내부 구현이 노출됨
2. "에러 처리 중 또 에러" 로 오독됨
3. "같은 사건 여러 레이어 로깅" 안티패턴과 결이 같음

### Case B: Repository 어댑터 — 인프라 예외 래핑 (향후)

```python
# 아직 미구현이지만 패턴은 확정
class PostgresERPRepository:
    def get(self, erp_id: str) -> ERP:
        try:
            row = self.session.query(...).filter_by(erp_id=erp_id).one()
        except sqlalchemy.exc.NoResultFound as e:
            raise NotFoundError(f"ERP {erp_id!r}") from None  # 구현 디테일 숨김
        except sqlalchemy.exc.OperationalError as e:
            raise RepositoryError("db unreachable") from e    # 원인 보존
```

`NoResultFound` → `NotFoundError`: 구현 디테일 변환 → `from None`
`OperationalError` → `RepositoryError`: 인프라 원인 보존 필요 → `from e`

---

## 4. `__context__` vs `__cause__` 내부 구조

```python
try:
    raise KeyError("x")
except KeyError:
    raise ValueError("y")           # __context__ = KeyError, __cause__ = None
                                     # __suppress_context__ = False

try:
    raise KeyError("x")
except KeyError as e:
    raise ValueError("y") from e    # __context__ = KeyError, __cause__ = KeyError
                                     # __suppress_context__ = True (cause가 있으니 context 표시 안 함)

try:
    raise KeyError("x")
except KeyError:
    raise ValueError("y") from None # __context__ = KeyError, __cause__ = None
                                     # __suppress_context__ = True (명시적으로 끊음)
```

| | `__context__` | `__cause__` | `__suppress_context__` | 트레이스백 표시 |
|---|---|---|---|---|
| 암묵적 | 원본 | None | False | "During handling of..." |
| `from e` | 원본 | e | True | "direct cause of..." |
| `from None` | 원본 | None | True | 원본 안 보임 |

`__context__`는 항상 설정된다 (Python 내부 동작). 차이는 표시 여부.

---

## 5. 요약

| 키워드 | 한 줄 |
|--------|------|
| 암묵적 | except 안에서 raise하면 자동 체이닝 — 의도적 변환에는 부적절 |
| `from e` | 인과 관계 명시 — 인프라 장애 래핑에 사용 |
| `from None` | 체이닝 끊기 — 내부 구현 디테일을 외부에 숨길 때 사용 |
| 판단 기준 | 트레이스백 독자에게 원본이 유용한 정보인가? |

---

## References

- [PEP 3134 — Exception Chaining and Embedded Tracebacks](https://peps.python.org/pep-3134/) — `__context__`, `__cause__` 도입
- [PEP 409 — Suppressing exception context](https://peps.python.org/pep-0409/) — `from None` 도입
- [Python docs — raise statement](https://docs.python.org/3/reference/simple_stmts.html#the-raise-statement)
- [error-handling-and-logging-principles.md](error-handling-and-logging-principles.md) — "잡은 사람이 한 번만 로그" 원칙
