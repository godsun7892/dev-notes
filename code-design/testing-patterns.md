# 테스트 환경 및 디버깅 패턴

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 빠른 단위 테스트와 느린 통합 테스트를 어떻게 조직하고, 의존성 교체와 결정론적 재현은 어떻게 설계하는가? |
| References | Apache Airflow (테스트 계층), FastAPI (DI), PettingZoo (결정론적 재현), factory_boy |

---

## 1. 테스트 계층 구조 (Airflow 패턴)

```
tests/
├── conftest.py              # 공유 fixtures (factory, config)
├── unit/                    # DB 없이, 함수 하나만 테스트, 0.001초
│   ├── conftest.py
│   ├── test_pricing.py
│   ├── test_validation.py
│   └── test_inventory.py
└── integration/             # DB 있고, 여러 함수 연동, 0.5초
    ├── conftest.py          # DB session fixture
    └── test_order_flow.py
```

실행:
```bash
pytest tests/unit/                                # unit만 (빠름)
pytest tests/integration/                         # integration만
pytest tests/unit/test_pricing.py::test_discount  # 함수 하나만
```

### 계층 분리 원칙
- **unit**: 순수 함수, 외부 I/O 없음, 밀리초 단위. CI에서 매번 실행
- **integration**: 실제 DB/네트워크 연결, 전체 시나리오. CI에서 PR마다
- **e2e**: 실제 배포된 환경 호출, 가장 느림. CI에서 nightly

Apache Airflow는 unit/integration/system 3단계로 분리. 대부분의 프로젝트는 unit/integration 2단계로 충분.

---

## 2. 의존성 교체 — Constructor Injection

설정 값(`if config.use_real_llm: ...`)으로 분기하지 말고, **진입점에서 의존성을 주입**한다.

```python
# 도메인 객체는 "어떤 전략이냐"를 모르고 동작
class OrderProcessor:
    def __init__(self, payment_gateway: PaymentGateway):
        self.payment_gateway = payment_gateway

# 테스트: test 진입점이 FakePaymentGateway 주입
processor = OrderProcessor(payment_gateway=FakePaymentGateway())

# 프로덕션: main 진입점이 StripeGateway 주입
processor = OrderProcessor(payment_gateway=StripeGateway(api_key="..."))
```

**settings 같은 스위칭 없음. 진입점이 다른 것뿐.**

이 패턴이 잘 들어맞는 조건: 도메인 코드가 외부 시스템을 인터페이스(`Protocol`)로만 알고 있음. FastAPI의 `Depends()`, NestJS의 DI 컨테이너, Spring의 `@Autowired`가 같은 원리.

---

## 3. 결정론적 재현 (PettingZoo 패턴)

비결정적 시스템(LLM, 외부 API, 강화학습)을 디버깅하려면 **이벤트 로그를 진실로 두고 재생**한다.

```
프로덕션: 외부 호출 → 행동 → 이벤트 로그에 기록
디버깅:   이벤트 로그 쿼리 → 스크립트 전략으로 변환 → 외부 호출 없이 재현
회귀테스트: 버그 발생 구간을 fixture로 저장 → 재발 방지
```

별도 replay 파일 불필요 — DB의 이벤트 로그가 곧 replay 데이터. 필요한 구간만 쿼리.

```sql
SELECT * FROM event_log
WHERE actor_id = 'actor_42' AND step BETWEEN 340 AND 350;
```

### Seed 테스트
동일 seed로 두 번 실행 → 결과 비교 → 비결정적 요소 탐지.

```python
def test_simulation_is_deterministic():
    result1 = run_simulation(seed=42)
    result2 = run_simulation(seed=42)
    assert result1 == result2
```

PettingZoo (multi-agent RL 라이브러리)는 모든 환경이 `seed()` API를 강제한다.

---

## 4. 테스트 데이터 팩토리 (factory_boy)

복잡한 도메인 객체를 매번 손으로 만들지 말고 팩토리로:

```python
import factory

class AccountFactory(factory.Factory):
    class Meta:
        model = Account

    account_id = factory.Sequence(lambda n: f"acc_{n}")
    role = "customer"
    cash = 200.0
    inventory = factory.LazyFunction(dict)

    class Params:
        rich = factory.Trait(cash=10000.0)
        broke = factory.Trait(cash=0.0)
        with_widget = factory.Trait(
            inventory=factory.LazyFunction(lambda: {"widget": 5})
        )
```

```python
AccountFactory()                                # 기본값
AccountFactory(rich=True, with_widget=True)     # 변형 조합
AccountFactory(role="vendor", cash=50)          # 직접 오버라이드
AccountFactory.create_batch(5)                  # 5개 한 번에
```

### Fixture vs Factory 사용 기준

```python
@pytest.fixture
def rich_customer():
    return AccountFactory(role="customer", with_widget=True)

def test_purchase(rich_customer):
    ...
```

- 자주 쓰는 조합 → fixture로 이름 부여
- 1회성/특수한 조합 → factory 직접 호출

factory_boy는 Django/SQLAlchemy/Pydantic 모두 지원. 다른 언어 동등물: Ruby `factory_bot`, JavaScript `fishery`.

---

## 5. DB 격리 (rollback per test)

통합 테스트에서 DB 상태가 테스트 간에 새지 않도록 각 테스트마다 트랜잭션 롤백:

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

- `scope="session"` (db_engine): 전체 테스트 세션에 1번만 생성 — 비싼 리소스
- 기본 scope (session): 매 테스트마다 새 트랜잭션, 끝나면 롤백 — 격리

unit 테스트는 DB 없이 순수 Python 객체. integration만 session fixture 사용.

### 실제 DB vs SQLite in-memory

| | SQLite in-memory | 실제 DB (PostgreSQL 등) |
|---|---|---|
| 속도 | 매우 빠름 | 느림 |
| SQL 호환성 | PG 기능 일부 미지원 (window function, CTE 일부) | 완전 호환 |
| 적합 | 빠른 피드백 루프 | 프로덕션 쿼리 검증 |

권장: 로컬 개발은 SQLite, CI는 testcontainers로 실제 DB.

---

## 6. 핵심 원칙 정리

| 원칙 | 한 줄 |
|------|------|
| 계층 분리 | unit (밀리초) / integration (초) / e2e (분) |
| DI | 설정 분기 X, 진입점에서 객체 주입 |
| 결정론 | seed + 이벤트 로그로 재현 |
| 팩토리 | 반복되는 객체 생성 패턴은 factory_boy |
| 격리 | 트랜잭션 롤백 또는 in-memory DB |

---

## References

- [Apache Airflow Testing Guide](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#testing-a-dag)
- [pytest fixtures docs](https://docs.pytest.org/en/stable/explanation/fixtures.html)
- [factory_boy documentation](https://factoryboy.readthedocs.io/)
- [PettingZoo (Multi-Agent RL)](https://pettingzoo.farama.org/)
- [testcontainers-python](https://testcontainers-python.readthedocs.io/) — 실제 DB 컨테이너로 통합 테스트
- [Cosmic Python — Testing chapter](https://www.cosmicpython.com/book/chapter_05_high_gear_low_gear.html)
- [Gary Bernhardt — Boundaries (Functional Core / Imperative Shell)](https://www.destroyallsoftware.com/talks/boundaries)
