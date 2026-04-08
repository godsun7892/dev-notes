# 테스트 환경 및 디버깅 패턴

| 항목 | 내용 |
|------|------|
| Date | 2026-04-08 |
| Context | AgentPit 경제 시뮬레이션 엔진 테스트 설계 |
| References | Apache Airflow (테스트 계층), FastAPI (DI), PettingZoo (결정론적 재현), factory_boy |

---

## 1. 테스트 계층 구조 (Airflow 패턴)

```
libs/economy_sim/tests/
├── conftest.py              # 공유 fixtures (factory, config)
├── unit/                    # DB 없이, 함수 하나만 테스트, 0.001초
│   ├── conftest.py
│   ├── test_spoilage.py
│   ├── test_npc_demand.py
│   └── test_trade.py
└── integration/             # DB 있고, 여러 함수 연동, 0.5초
    ├── conftest.py          # DB session fixture
    └── test_round_cycle.py
```

실행:
```bash
pytest tests/unit/              # unit만 (빠름)
pytest tests/integration/       # integration만
pytest tests/unit/test_spoilage.py::test_wheat_spoils  # 함수 하나만
```

---

## 2. 의존성 교체 — Constructor Injection

```python
# RoundEngine은 "어떤 전략이냐"를 모르고 동작
class RoundEngine:
    def __init__(self, strategies: dict[str, AgentStrategy]):
        self.strategies = strategies

# 테스트: test_round.py가 진입점 → ScriptedStrategy 주입
engine = RoundEngine(strategies={"mill": ScriptedStrategy({1: [Buy("wheat", 3)]})})

# 프로덕션: main.py가 진입점 → LLMStrategy 주입
engine = RoundEngine(strategies={"mill": LLMStrategy(api_key="...")})
```

settings 같은 스위칭 없음. 진입점이 다른 것뿐.

---

## 3. 결정론적 재현 (PettingZoo 패턴)

프로덕션에서 이벤트 로그(action_log)를 DB에 기록. 디버깅 시 해당 구간 쿼리 → ScriptedStrategy로 변환 → LLM 없이 재현.

```
프로덕션: LLM 호출 → 행동 → 이벤트 로그에 기록
디버깅:   이벤트 로그 쿼리 → ScriptedStrategy → LLM 없이 재현
회귀테스트: 버그 발생 구간을 fixture로 저장 → 재발 방지
```

별도 replay 파일 불필요 — DB의 이벤트 로그가 곧 replay 데이터. 필요한 구간/에이전트만 쿼리.

```sql
SELECT * FROM event_log 
WHERE agent_id = 'agent_42' AND round BETWEEN 340 AND 350;
```

seed 테스트: 동일 seed로 두 번 실행 → 결과 비교 → 비결정적 요소 탐지.

---

## 4. 테스트 데이터 팩토리 (factory_boy)

```python
class ERPFactory(factory.Factory):
    class Meta:
        model = ERP

    agent_id = factory.Sequence(lambda n: f"agent_{n}")
    role = "wheat_farm"
    cash = 200.0
    inventory = factory.LazyFunction(dict)

    class Params:
        rich = factory.Trait(cash=10000.0)
        broke = factory.Trait(cash=0.0)
        with_wheat = factory.Trait(
            inventory=factory.LazyFunction(
                lambda: {"wheat": [ResourceBatch("wheat", 6, produced_at=1)]}
            )
        )
```

```python
ERPFactory()                              # 기본값
ERPFactory(rich=True, with_wheat=True)    # 변형 조합
ERPFactory(role="bakery", cash=50)        # 직접 오버라이드
ERPFactory.create_batch(5)                # 5개 한 번에
```

conftest fixture 안에서 factory 사용:
```python
@pytest.fixture
def wheat_farm():
    return ERPFactory(role="wheat_farm", with_wheat=True)
```

자주 쓰는 조합 → fixture. 특수한 조합 → factory 직접 호출.

---

## 5. DB 격리 (rollback per test)

```python
@pytest.fixture(scope="session")
def db_engine():
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    yield engine
    engine.dispose()

@pytest.fixture
def session(db_engine):
    conn = db_engine.connect()
    txn = conn.begin()
    session = Session(bind=conn)
    yield session
    session.close()
    txn.rollback()    # 매 테스트마다 깨끗한 상태
    conn.close()
```

unit은 DB 없이 순수 Python 객체. integration만 session fixture 사용.
