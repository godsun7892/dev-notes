# Shallow Copy vs Deep Copy — 언제 뭘 써야 하나

Python에서 객체를 "복사"하는 세 가지 방법의 차이와,
immutable 패턴에서 왜 shallow copy가 정답인지를 정리한다.

---

## 1. 세 가지 복사 방법

### 할당 (복사가 아님)

```python
a = {"x": [1, 2]}
b = a                  # 같은 dict를 가리키는 이름이 하나 더 생긴 것
b["y"] = [3]
print(a)               # {"x": [1, 2], "y": [3]}  ← a도 바뀜
```

메모리:
```
a ──→ {"x": [1, 2]}
b ──↗                   ← 하나의 객체, 이름만 둘
```

### Shallow Copy

```python
a = {"x": [1, 2], "y": [3, 4]}
b = dict(a)            # 또는 a.copy(), copy.copy(a)

b["z"] = [5]           # a에 안 생김 ✅  (dict 껍데기가 다르니까)
b["x"].append(999)     # a["x"]도 [1, 2, 999] ❌  (안의 list는 공유)
```

메모리:
```
a ──→ dict { "x": ──→ [1, 2]     ← 같은 리스트
b ──→ dict { "x": ──↗
             "y": ──→ [3, 4]     ← 같은 리스트
             "y": ──↗
```

**껍데기(dict)는 새로 만들지만, 안에 담긴 객체(list)는 공유한다.**
안의 객체를 변경하면 양쪽 모두 영향받는다.

### Deep Copy

```python
import copy
a = {"x": [1, 2], "y": [3, 4]}
b = copy.deepcopy(a)

b["x"].append(999)
print(a["x"])          # [1, 2]  ✅  (완전 별개)
```

메모리:
```
a ──→ dict { "x": ──→ [1, 2]       ← a만의 list
b ──→ dict { "x": ──→ [1, 2]       ← b만의 list (별도 객체)
```

**전체 트리를 재귀적으로 복제한다.** 안전하지만 느리다.

---

## 2. 비용 비교

에이전트 10명, 각각 ResourceBatch 8개를 가진 ERP dict 기준:

| 방식 | 1회 소요 | 배수 |
|------|---------|------|
| `dict(erps)` shallow | ~0.12 μs | 1x |
| `{k: replace(v) ...}` | ~16 μs | 130x |
| `copy.deepcopy(erps)` | ~534 μs | **4,300x** |

deepcopy가 느린 이유: ERP → Inventory → holdings (dict) → tuple → ResourceBatch
전체 객체 트리를 탐색하면서 하나하나 새 인스턴스를 만든다.

```
deepcopy(erps)
├─ 새 dict 생성
├─ erps["e0"] → deepcopy(ERP)
│  ├─ deepcopy(Inventory)
│  │  ├─ deepcopy(holdings dict)
│  │  │  ├─ deepcopy(tuple of ResourceBatch)
│  │  │  │  ├─ deepcopy(ResourceBatch) × 4
│  │  │  │  └─ ...
│  │  │  └─ ...
│  │  └─ ...
│  └─ ...
├─ erps["e1"] → deepcopy(ERP)
│  └─ ... (같은 재귀)
└─ × 10명
```

---

## 3. Immutable 객체면 shallow copy로 충분한 이유

핵심 질문: **shallow copy의 "안의 객체 공유" 문제가 발생하려면?**

→ 공유 중인 객체를 **변경(mutate)**해야 한다.

frozen dataclass는 변경이 불가능하다:

```python
@dataclass(frozen=True)
class ERP:
    cash: int = 200

erp = ERP(...)
erp.cash = 300    # FrozenInstanceError!
```

값을 바꾸려면 `replace()`로 **아예 새 객체**를 만들어야 한다:

```python
new_erp = replace(erp, cash=300)
# erp와 new_erp는 완전히 별개 인스턴스
# id(erp) != id(new_erp)
```

따라서:

```python
original = {"a": erp_a}
copied = dict(original)          # shallow copy. erp_a를 공유

copied["a"] = replace(erp_a, cash=300)   # 새 인스턴스로 교체
# original["a"]는 여전히 erp_a (cash=200)  ← 안전
```

**공유해도 변경이 불가능하니 문제가 발생하지 않는다.**
변경하려면 새 객체를 만들어 dict에 대입하는데,
shallow copy는 dict 껍데기가 별개이므로 원본 dict에 영향이 없다.

---

## 4. Shallow Copy가 깨지는 경우: Frozen 안의 Mutable

frozen dataclass에 mutable 필드가 있으면 이야기가 달라진다:

```python
@dataclass(frozen=True)
class Broken:
    data: dict[str, int]    # ← mutable dict

obj = Broken(data={"a": 1})
obj.data["a"] = 999          # FrozenInstanceError 아님! dict 자체를 변경한 거라
                              # dataclass 필드 재할당이 아님
```

frozen은 `obj.data = new_dict`를 막지, `obj.data["key"] = val`은 못 막는다.
dict 객체의 내부 변경이기 때문이다.

이 경우 shallow copy로 공유하면:
```python
copied = dict(original)
# copied["x"]와 original["x"]가 같은 Broken 인스턴스
# Broken.data도 같은 dict
copied["x"].data["a"] = 999   # original["x"].data도 오염!
```

**해결책**:
- frozen dataclass에는 immutable 타입만: `tuple`, `frozenset`, `MappingProxyType`
- 또는 `__post_init__`에서 방어 복사

---

## 5. 판단 기준 요약

```
컬렉션 안의 객체가 immutable인가?
├─ YES → shallow copy (dict(), list(), .copy())
│        deepcopy는 낭비. 공유해도 변경 불가하니 안전.
│
└─ NO  → 상황에 따라:
         ├─ 읽기만 할 거면 → shallow copy OK
         ├─ 안의 객체를 변경할 거면 → deep copy 또는 수동 복사
         └─ 성능이 중요하면 → immutable로 바꾸는 게 근본 해결
```

---

## 6. economy_sim에서의 적용

economy_sim의 모든 도메인 객체는 `frozen=True`다.
따라서 모든 엔진에서 컬렉션 복사는 shallow copy를 사용한다.

| 엔진 | 복사 대상 | 방식 |
|------|----------|------|
| `settle_npc_sales` | `erps: dict[str, ERP]` | `dict(erps)` |
| `with_consumed` | `holdings: dict[str, tuple[...]]` | dict comprehension |
| `with_added` | `holdings` | `dict(self.holdings)` |

**주의**: `ProductionOrder.outputs`와 `Recipe.inputs/outputs`는 frozen dataclass 안의
plain dict라서 4번 섹션의 위험에 해당한다. 현재는 엔진이 `dict(recipe.inputs)`로
방어 복사하고 있지만, 구조적으로 tuple이나 `MappingProxyType`로 전환하는 게 안전하다.

## 관련 문서

- [docs/design/immutable-copy-strategy.md](../../docs/design/immutable-copy-strategy.md) — 실무 가이드
- [dev-notes/code-design/functional-core-engine-pattern.md](functional-core-engine-pattern.md) — immutable 엔진 규약
