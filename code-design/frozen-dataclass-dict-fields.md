# Frozen Dataclass + Dict 필드 — 구멍과 해법

`@dataclass(frozen=True)`에 `dict` 필드를 넣으면 생기는 문제,
업계가 이걸 어떻게 다루는지, 그리고 우리의 선택을 정리한다.

---

## 1. 문제: frozen은 dict 내부를 못 막는다

```python
@dataclass(frozen=True)
class Recipe:
    inputs: dict[str, int]

recipe = Recipe(inputs={"flour": 2, "butter": 1})
recipe.inputs = {}              # FrozenInstanceError ✅ 막힘
recipe.inputs["flour"] = 999    # 통과 ❌ 안 막힘
```

`frozen=True`는 **필드 재할당**(`obj.field = x`)만 막는다.
필드가 가리키는 dict 객체의 **내부 변경**(`obj.field[key] = val`)은 막지 못한다.
dict 자체는 mutable 객체이기 때문이다.

이건 Python dataclass의 알려진 한계이며, Pydantic과 attrs도 동일하다.

---

## 2. 왜 문제인가

frozen dataclass를 쓰는 이유는 "이 객체는 불변"이라는 **계약**을 선언하는 것이다.
dict 필드가 변경 가능하면 그 계약이 거짓이 된다.

```python
# 엔진이 Recipe를 받아서 처리
result = start_production(inventory=inv, scenario=scenario, ...)

# 그 뒤 다른 코드가 같은 recipe를 변경
scenario.recipes["baking"].inputs["flour"] = 0  # 통과!

# 다음 호출에서 엔진이 오염된 recipe를 받음
result2 = start_production(...)  # flour 0개로 빵이 만들어짐
```

frozen이라 안전하다고 믿고 방어 코드를 안 넣은 곳에서 터진다.
그리고 **원인 추적이 어렵다** — frozen인데 왜 값이 바뀌었지? 라는 혼란.

---

## 3. 업계 실제 패턴 — 5가지

### 패턴 A: `MappingProxyType` 감싸기

**CPython `dataclasses.py` — 자기 자신의 메타데이터 보호:**
```python
# Lib/dataclasses.py (CPython 소스)
self.metadata = types.MappingProxyType(metadata)
```

**attrs — 동일한 방식:**
```python
# src/attr/_make.py
metadata = types.MappingProxyType(dict(metadata))
```

**Home Assistant — frozen dataclass에서 사용:**
```python
@dataclass(frozen=True, kw_only=True)
class ConfigSubentry:
    data: MappingProxyType[str, Any]
    subentry_id: str
    title: str
```

특징:
- 읽기 API가 dict와 동일 (`[]`, `.get()`, `.keys()`, `in`, 반복)
- 쓰기 시도 시 `TypeError` 즉시 발생
- **not hashable** — dict key/set member로 못 씀
- O(1) lookup, 생성 비용 ~1.7x

### 패턴 B: dict 상속 + 변경 메서드 차단

**SQLAlchemy — C 확장까지 만듦:**
```python
class immutabledict(dict):
    def __setitem__(self, key, value): raise TypeError
    def __delitem__(self, key):        raise TypeError
    def clear(self):                   raise TypeError
    # pop, update, setdefault 전부 차단
    
    def union(self, *dicts):  return immutabledict(merged)
    def copy(self):           return self  # immutable이니 자기 자신 리턴
```

**Flask/Werkzeug:**
```python
class Flask:
    default_config = ImmutableDict({
        "DEBUG": None,
        "TESTING": False,
        # ...
    })
```

특징:
- `isinstance(x, dict)` 통과 — 기존 dict 기대하는 코드와 호환
- SQLAlchemy는 성능 때문에 C 확장도 유지
- 자체 구현 부담

### 패턴 C: frozen 클래스에 dict를 아예 안 넣음

**Sentry:**
```python
@dataclass(frozen=True)
class SymbolicatorTaskKind:
    platform: SymbolicatorPlatform
    is_reprocessing: bool = False
```

**Apache Airflow:**
```python
@attrs.define(frozen=True)
class AssetUniqueKey:
    name: str
    uri: str
```

**Cosmic Python (DDD 교과서):**
```python
@dataclass(frozen=True)
class OrderLine:
    orderid: str
    sku: str
    qty: int
```

특징:
- **가장 많은 프로젝트가 쓰는 방식**
- frozen에는 스칼라만, dict가 필요하면 frozen을 안 건다
- 문제를 근본적으로 회피

### 패턴 D: 정적 타입으로만 보호

**Dropbox (400만 줄 Python) / PEP 705 (ReadOnly):**
```python
@dataclass(frozen=True)
class Recipe:
    inputs: Mapping[str, int]  # Mapping은 읽기 전용 인터페이스
```

`Mapping`으로 어노테이션하면 mypy/pyright가 `recipe.inputs["key"] = val`을
정적으로 잡아준다. 런타임 보호는 없지만, CI에서 타입 체크를 돌리면 충분하다는 철학.

특징:
- 런타임 비용 0
- 타입 체커 통과하면 안전
- 타입 체커를 안 쓰는 환경에서는 무방비

### 패턴 E: Python 3.15 `frozendict` (2026.10 예정)

```python
fd = frozendict({"flour": 2, "butter": 1})
fd["flour"]          # 2 — 읽기 OK
fd["flour"] = 999    # TypeError — 쓰기 차단
hash(fd)             # 동작 — hashable
```

CPython `dataclasses.py`가 이미 내부적으로 전환 완료:
```python
_EMPTY_METADATA = frozendict()  # 기존: MappingProxyType({})
```

특징:
- hashable + 불변 + O(1) lookup
- stdlib 내장, 의존성 없음
- 아직 alpha. stable은 2026.10

---

## 4. 패턴별 비교

| | MappingProxyType | dict 상속 (SQLAlchemy식) | dict 회피 | 정적 타입만 | frozendict (3.15) |
|---|---|---|---|---|---|
| **런타임 보호** | O | O | 해당 없음 | X | O |
| **hashable** | X | X | - | - | O |
| **읽기 API** | dict 동일 | dict 동일 | - | dict 동일 | dict 동일 |
| **isinstance(x, dict)** | X | O | - | - | X |
| **구현 비용** | `__post_init__` 몇 줄 | 클래스 하나 | 모델 재설계 | 어노테이션만 | `__post_init__` 몇 줄 |
| **외부 의존** | 없음 (stdlib) | 없음 | 없음 | mypy/pyright | 없음 (3.15+) |
| **쓰는 곳** | CPython, attrs, HA | SQLAlchemy, Flask | Sentry, Airflow | Dropbox | CPython (이미 전환) |

---

## 5. 우리의 선택: MappingProxyType → frozendict 전환 경로

economy_sim의 상황:
- frozen dataclass에 `dict[str, int]` 필드가 있다 (Recipe, ProductionOrder, Scenario)
- 엔진이 `recipe.inputs["resource_id"]`로 O(1) 접근한다 → tuple 전환은 비용 큼
- hashable이 필요하지 않다 (도메인 모델, dict key로 안 씀)
- 소비자(Worker/GM)가 아직 0줄이지만 곧 붙는다

**지금: `Mapping` 어노테이션 + `MappingProxyType`**

```python
from types import MappingProxyType
from collections.abc import Mapping

@dataclass(frozen=True)
class Recipe:
    name: str
    inputs: Mapping[str, int]
    outputs: Mapping[str, int]
    duration: int

    def __post_init__(self) -> None:
        object.__setattr__(self, "inputs", MappingProxyType(dict(self.inputs)))
        object.__setattr__(self, "outputs", MappingProxyType(dict(self.outputs)))
```

- 호출자는 plain dict를 넘겨도 됨 (Mapping이 dict의 상위 타입)
- 읽기는 그대로: `recipe.inputs["flour"]`, `.get()`, `in`
- 쓰기 시도: `TypeError` 즉시 발생
- `replace()` 시 `__post_init__` 재실행 → 새 인스턴스도 보호됨
- mypy/pyright가 `Mapping`에 대한 쓰기를 정적으로도 잡아줌

**Python 3.15 이후: `frozendict`로 전환**

```python
@dataclass(frozen=True)
class Recipe:
    name: str
    inputs: frozendict[str, int]
    outputs: frozendict[str, int]
    duration: int

    def __post_init__(self) -> None:
        object.__setattr__(self, "inputs", frozendict(self.inputs))
        object.__setattr__(self, "outputs", frozendict(self.outputs))
```

`__post_init__` 안의 `MappingProxyType(dict(...))` → `frozendict(...)` 한 줄 교체.
hashable도 공짜로 얻는다.

---

## 6. 적용 대상

| 모델 | 필드 | 현재 | 전환 후 |
|------|------|------|--------|
| `Recipe` | `inputs`, `outputs` | `dict[str, int]` | `Mapping[str, int]` |
| `ProductionOrder` | `outputs` | `dict[str, int]` | `Mapping[str, int]` |
| `Scenario` | `resources`, `recipes`, `roles`, `npc_intermediate_prices` | `dict[str, ...]` | `Mapping[str, ...]` |
| `Inventory` | `holdings` | `dict[str, tuple[...]]` | `Mapping[str, tuple[...]]` |
| `GameEvent` | `properties` | `dict[str, Any]` | `Mapping[str, Any]` |
| `GameConfig` | (스칼라만) | - | 변경 없음 |

---

## 관련 문서

- [dev-notes/code-design/shallow-vs-deep-copy.md](shallow-vs-deep-copy.md) — shallow/deep copy 이론
- [docs/design/immutable-copy-strategy.md](../../docs/design/immutable-copy-strategy.md) — 엔진의 복사 전략
- [dev-notes/code-design/functional-core-engine-pattern.md](functional-core-engine-pattern.md) — immutable 엔진 규약

## 참고 소스

- [CPython dataclasses.py](https://github.com/python/cpython/blob/main/Lib/dataclasses.py) — MappingProxyType 사용
- [attrs _make.py](https://github.com/python-attrs/attrs/blob/main/src/attr/_make.py) — 동일 패턴
- [SQLAlchemy immutabledict](https://github.com/sqlalchemy/sqlalchemy/blob/main/lib/sqlalchemy/util/_immutabledict_cy.py) — dict 상속 패턴
- [Home Assistant config_entries.py](https://github.com/home-assistant/core/blob/dev/homeassistant/config_entries.py) — frozen dataclass + MappingProxyType
- [PEP 814 — frozendict](https://peps.python.org/pep-0814/) — Python 3.15 builtin
- [Pydantic frozen 한계](https://github.com/pydantic/pydantic/issues/12361) — shallow freeze만
