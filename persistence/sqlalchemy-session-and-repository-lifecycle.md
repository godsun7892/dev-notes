# SQLAlchemy async — Session 과 Repository 의 수명

| 항목 | 내용 |
|------|------|
| 핵심 질문 | Repository 를 싱글톤으로 한 번 만들어두고 계속 재사용해도 되나? |
| 결론 | **안 된다.** Repository 는 Session 의 수명과 같이 간다. 싱글톤은 Engine / session_factory 층에만. |

---

## 1. 왜 헷갈리는가 — 생태계마다 다르다

### Java Spring / .NET 쪽

`@Repository`, `DbContext` 는 **싱글톤 또는 scoped bean** 으로 한 번 등록해두고 계속 쓴다. 그게 가능한 이유:

- 프레임워크가 뒤에서 **EntityManager/DbContext 를 요청별 프록시** 로 갈아끼워줌
- 개발자는 싱글톤처럼 쓰지만 실제 DB 연결·트랜잭션은 요청마다 새로
- 마법 덕에 "repo = 싱글톤" 이 자연스러워 보임

### Python SQLAlchemy async

이런 프록시 마법이 **없다**. Repository 가 Session 을 생성자로 직접 받는다:

```python
class PgOfferRepository:
    def __init__(self, session: AsyncSession) -> None:
        self._session = session
```

즉 Repository = Session 을 쥔 얇은 손잡이. **Session 이 공유 불가능하면 Repository 도 공유 불가능**.

---

## 2. AsyncSession 이 공유되면 안 되는 이유

SQLAlchemy 공식 docs 가 명확히 못박는다: **"세션 = 하나의 논리적 작업 단위. 여러 요청/코루틴 공유 금지"**. 이유는 Session 이 들고 있는 상태:

| Session 내부 상태 | 설명 | 공유 시 사고 |
|---|---|---|
| Identity map | 이미 로드된 row 캐시 (PK → 객체) | A 가 load 한 row 를 B 가 수정하면 양쪽이 섞임 |
| Pending changes | 아직 flush 안 된 add/delete/dirty | A 의 pending INSERT 가 B 의 commit 에 딸려 들어감 |
| Transaction state | `begin()` 으로 연 TX 하나 | A 의 rollback 이 B 의 작업도 전부 날림 |
| Connection | 풀에서 빌려온 연결 하나 | A 의 FOR UPDATE 락이 B 를 영문도 모르게 블록 |

즉 "싱글톤 Repository = 싱글톤 Session = 동시성 불가".

---

## 3. 실제로 싱글톤인 것들

SQLAlchemy async 의 레이어별 수명:

| 레이어 | 수명 | 개수 | 비용 |
|---|---|---|---|
| `AsyncEngine` (연결 풀 보유) | 프로세스 시작 ~ 종료 | **싱글톤 1개** | 생성 비쌈 (풀 초기화, TCP handshake) |
| `async_sessionmaker` | 프로세스 시작 ~ 종료 | **싱글톤 1개** | 저렴 (engine 래핑만) |
| `AsyncSession` | 한 번의 UoW (tool 호출 1회) | **매번 새로** | 풀에서 연결 1개 빌려오는 수준 |
| `Pg*Repository(session)` | session 과 동일 | **매번 새로** | `self._session = session` — 사실상 0 |

싱글톤 최적화 욕심을 낼 가치가 있는 건 **Engine** 하나뿐. 나머지는 매번 새로 만들어도 측정 안 되는 비용.

우리 `apps/worker/src/worker/adapters/postgres/engine.py` 가 이 구조를 따른다:

```python
_engine: AsyncEngine | None = None
_session_factory: async_sessionmaker[AsyncSession] | None = None

def get_engine() -> AsyncEngine:
    global _engine, _session_factory
    if _engine is None:
        _engine = create_async_engine(...)
        _session_factory = async_sessionmaker(_engine, expire_on_commit=False)
    return _engine
```

Engine / session_factory 는 lazy 싱글톤. 실제 호출 경로에선 `session_factory()` 로 **매번 새 session** 을 열고, 그 session 으로 **매번 새 Repository** 를 조립한다.

---

## 4. 수명 매핑 그림

```
프로세스 수명 ────────────────────────────────────────────────
  AsyncEngine (연결 풀)                              싱글톤
  async_sessionmaker                                 싱글톤
────────────────────────────────────────────────────────────
  tool 호출 1회 ─ AsyncSession                       매번 새로
                   └ PgOfferRepository(session)      매번 새로
                   └ PgERPRepository(session)        매번 새로
                   └ PgTradeRepository(session)      매번 새로
```

Tool 한 번 실행 = Session 한 개 + Repository 여러 개. 같은 session 을 공유하는 repo 들은 **같은 트랜잭션**, **같은 identity map** 안에 있어서 multi-repo 원자성이 자연스럽게 나온다 (ADR-014 §D3 "caller 가 TX 소유" 와 일치).

---

## 5. 예외 — stateless repo 라면 싱글톤도 가능

만약 repo 가 session 을 쥐지 않고 **메서드 인자로 받는** 형태라면:

```python
class OfferRepoStateless:
    @staticmethod
    async def create(session: AsyncSession, offer: Offer) -> None:
        session.add(OfferRow(...))
        await session.flush()
```

이러면 repo 는 상태가 없어서 싱글톤이든 static 메서드 모음이든 상관 없다. 트레이드오프:

| 기준 | 생성자 주입 (우리 방식) | 인자 주입 (stateless) |
|---|---|---|
| 호출 코드 | `repo.create(offer)` — 깔끔 | `repo.create(session, offer)` — session 매번 명시 |
| multi-repo 합성 | `OfferRepo(s), TradeRepo(s)` 동시 보유 가능 | 메서드마다 session 넣기 |
| Testability (mock) | 생성자에 fake session 주입 | static 이라 monkeypatch 필요 |
| Cosmic Python 추천 | ✅ 정석 | ❌ anti |

Cosmic Python / SQLAlchemy docs 예제 / advanced-alchemy 전부 **생성자 주입** 을 쓴다. stateless 는 함수형 스타일 선호하는 팀에서 가끔 보이는 정도.

---

## 6. 비유

```
Engine           = 수도 회사 (싱글톤)
session_factory  = 수도꼭지 (싱글톤)
AsyncSession     = 물 한 컵          ← 작업마다 새로
Pg*Repository    = 컵 손잡이          ← 물 컵과 한 몸
```

손잡이만 싱글톤으로 들고 물을 재탕할 순 없다. Java Spring 은 뒤에서 컵을 몰래 갈아끼워주는 프록시가 있고, Python async 에는 없어서 **매번 새로 조립** 이 정석이다.

---

## 7. 우리 코드에서의 실제 흐름

`apps/worker/src/worker/tools/offer_tools.py` 의 `accept_offer` 가 전형적인 예:

```python
async def accept_offer(..., session_factory):
    # Preview (read-only) ─ session 1
    async with session_factory() as preview_session:
        offer_repo = PgOfferRepository(preview_session)
        offer = await offer_repo.get(offer_id)
        ...

    # TX ─ session 2 (별개 session, 별개 TX)
    async with session_factory() as session, session.begin():
        offer_repo = PgOfferRepository(session)   # ← 또 새로 생성
        erp_repo   = PgERPRepository(session)
        trade_repo = PgTradeRepository(session)
        await erp_repo.save(...)
        await offer_repo.mark_accepted(...)
        await trade_repo.create(...)
    # exit = commit
```

preview 용 session 과 TX 용 session 이 **완전히 다른 인스턴스**. preview 에서 들어있던 identity map / pending state 는 TX session 에 새지 않는다. Repository 도 각 session 에 맞춰 매번 재조립.

이 구조가 10만 concurrent agent (ADR-012) 에서 문제없이 스케일하는 근거 — 각 tool 호출이 자기 session/repo 짝 하나만 쓰고 끝이라, 코루틴 간 상태 간섭이 0.

---

## 참고

- [SQLAlchemy Session Basics — "Session is not thread-safe"](https://docs.sqlalchemy.org/en/20/orm/session_basics.html)
- [AsyncIO Extension — "the AsyncSession object is a sync style"](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- Cosmic Python, Chapter 6 "Unit of Work Pattern" — Repository 가 session 을 보유하는 정석 구조
- [ADR-014 §D3](../../docs/adr/014-state-oriented-persistence-pivot.md) — caller 가 TX 경계 소유
