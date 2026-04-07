# Python 모노레포 구조 리서치

> 연구 목적: AgentPit처럼 여러 서비스(web, worker, gamemaster)가 공유 코드를 사용하는 Python 프로젝트의 최적 구조 파악

---

## 1. 실제 프로젝트 디렉토리 트리 (GitHub에서 직접 확인)

### 패턴 A: `apps/` + `libs/` (carderne/postmodern-mono)

**uv workspace 기반, 가장 현대적인 패턴.**

```
postmodern-mono/
├── pyproject.toml          # 루트: workspace 정의만
├── uv.lock                 # 단일 lockfile (전체 공유)
├── apps/
│   ├── mycli/
│   │   ├── pyproject.toml  # 개별 패키지 설정
│   │   ├── postmodern/     # 네임스페이스 패키지
│   │   │   └── mycli/
│   │   └── tests/
│   └── server/
│       ├── Dockerfile
│       ├── pyproject.toml
│       ├── postmodern/
│       │   └── server/
│       └── tests/
└── libs/
    └── greeter/
        ├── pyproject.toml
        ├── postmodern/
        │   └── greeter/
        └── tests/
```

루트 `pyproject.toml` 핵심:
```toml
[tool.uv.workspace]
members = ["libs/*", "apps/*"]   # glob으로 자동 인식
```

서비스의 `pyproject.toml`:
```toml
[project]
dependencies = ["postmodern-greeter"]  # 공유 라이브러리 의존

[tool.uv.sources]
postmodern-greeter = { workspace = true }  # PyPI가 아닌 워크스페이스에서 해결
```

**특징:**
- 네임스페이스 패키징 (`postmodern.server`, `postmodern.greeter`)
- 각 패키지가 독립 `pyproject.toml` 보유
- `workspace = true`로 내부 의존성 선언

---

### 패턴 B: `services/` + `packages/` (mvoss02/uv_workspaces_example)

**서비스와 라이브러리를 명시적으로 분리.**

```
uv_workspaces_example/
├── pyproject.toml
├── uv.lock
├── main.py
├── packages/                        # 공유 라이브러리들
│   ├── my_package_1/
│   │   ├── pyproject.toml
│   │   └── src/
│   │       └── my_package_1/
│   │           ├── __init__.py
│   │           └── lib.py
│   └── my_package_2/
│       ├── pyproject.toml
│       └── src/
│           └── my_package_2/
└── services/                        # 배포 가능한 서비스들
    ├── data_ingestion/
    │   ├── pyproject.toml
    │   └── data_ingest/
    │       └── main.py
    └── data_preprocessing/
        ├── pyproject.toml
        └── data_pre/
            └── main.py
```

루트 `pyproject.toml`:
```toml
[tool.uv.workspace]
members = [
    "packages/my_package_1",
    "packages/my_package_2",
    "services/data_ingestion",
    "services/data_preprocessing"
]

[tool.uv.sources]
my-package-1 = { workspace = true }
my-package-2 = { workspace = true }
data-ingest = { workspace = true }
data-pre = { workspace = true }
```

---

### 패턴 C: `services/` + `packages/` (matanby/python-monorepo-template)

**서비스 생성 스크립트까지 포함한 실전 템플릿.**

```
python-monorepo-template/
├── pyproject.toml
├── Makefile
├── packages/          # 공유 패키지들 (여기서 생성)
│   └── .gitkeep
├── services/          # 서비스들 (여기서 생성)
│   └── .gitkeep
├── scripts/
│   ├── new-package.sh   # 공유 패키지 생성 스크립트
│   └── new-service.sh   # 서비스 생성 스크립트
└── templates/
    ├── package/         # 패키지 템플릿
    │   ├── pyproject.toml
    │   └── src/package_name/
    └── service/         # 서비스 템플릿
        ├── Dockerfile
        ├── pyproject.toml
        └── src/service_name/
```

---

### 패턴 D: `libs/` (tweag/python-monorepo-example)

**Pants 빌드 + 네임스페이스 패키지.**

```
python-monorepo-example/
├── pyproject.toml
├── tox.ini
├── libs/
│   ├── base/
│   │   ├── pyproject.toml
│   │   ├── mycorp/            # 네임스페이스
│   │   │   └── base/
│   │   │       ├── __init__.py
│   │   │       └── adder2.py
│   │   └── tests/
│   └── fancy/
│       ├── pyproject.toml
│       ├── mycorp/
│       │   └── fancy/
│       │       ├── __init__.py
│       │       └── adder3.py  # mycorp.base.adder2를 import
│       └── tests/
└── templates/
    └── pylibrary/             # cookiecutter 템플릿
```

**핵심:** `mycorp`이 네임스페이스 역할. `mycorp.base`, `mycorp.fancy`로 import.

---

## 2. 실제 게임/시뮬레이션 플랫폼 구조

### NegMAS (협상 멀티에이전트 시뮬레이션)

**단일 패키지, 도메인별 서브패키지로 분리.**

```
negmas/                          # 단일 저장소
└── src/
    └── negmas/                  # 하나의 큰 패키지
        ├── __init__.py
        │
        │  # === 핵심 도메인 모듈 ===
        ├── mechanisms.py        # 협상 프로토콜/메커니즘
        ├── protocols.py         # 프로토콜 인터페이스
        │
        │  # === 도메인 서브패키지들 ===
        ├── negotiators/         # 협상 에이전트
        │   ├── negotiator.py
        │   ├── controller.py
        │   ├── controlled.py
        │   ├── modular.py
        │   ├── simple.py
        │   ├── helpers.py
        │   └── components/      # 더 깊은 하위 분류
        │
        ├── situated/            # 시뮬레이션 월드
        │   ├── world.py         # 핵심 World 클래스
        │   ├── agent.py         # 시뮬레이션 내 에이전트
        │   ├── contract.py      # 계약
        │   ├── breaches.py      # 계약 위반
        │   ├── awi.py           # Agent-World Interface
        │   ├── action.py
        │   ├── entity.py
        │   └── helpers.py
        │
        ├── outcomes/            # 협상 결과
        ├── preferences/         # 선호도/효용 함수
        ├── scenarios/           # 시나리오 정의
        ├── tournaments/         # 토너먼트 시스템
        │
        │  # === 특정 프로토콜 구현 ===
        ├── sao/                 # Stacked Alternating Offers
        ├── gb/                  # General Bargaining
        │
        │  # === 인프라 ===
        ├── helpers/
        ├── models/
        ├── plots/
        ├── scripts/
        ├── types/
        ├── config.py
        ├── common.py
        ├── events.py
        ├── exceptions.py
        ├── serialization.py
        └── tests/
```

### SCML (Supply Chain Management League) - NegMAS 기반 게임

**게임별 코드는 별도 저장소, negmas를 외부 의존성으로 사용.**

```
scml/                            # 별도 저장소
└── src/
    └── scml/
        ├── __init__.py
        ├── common.py            # SCML 전역 공유 코드
        ├── utils.py
        ├── runner.py            # 게임 실행기
        ├── experiment.py        # 실험 관리
        ├── cli.py               # CLI
        │
        │  # === 게임 버전별 서브패키지 ===
        ├── oneshot/             # OneShot 게임 모드
        │   ├── world.py         # World 구현
        │   ├── agent.py         # 에이전트 기본 클래스
        │   ├── awi.py           # Agent-World Interface
        │   ├── ufun.py          # 유틸리티 함수
        │   ├── policy.py
        │   ├── agents/          # 빌트인 에이전트들
        │   └── rl/              # 강화학습 관련
        │
        ├── std/                 # 표준 게임 모드
        ├── scml2019/            # 2019년 버전
        ├── scml2020/            # 2020년 버전
        └── vendor/              # 벤더링된 외부 코드
```

**핵심 관찰:** NegMAS(프레임워크) vs SCML(게임)이 별도 저장소. 
게임이 프레임워크를 `pyproject.toml`의 dependency로 참조.

---

## 3. 산업계 사례: Opendoor

**실제 프로덕션 모노레포 구조.**

```
opendoor-monorepo/
├── projects/                     # 배포 가능한 서비스들
│   └── my-service/
│       ├── Dockerfile
│       ├── pyproject.toml
│       ├── my_service/           # 서비스 코드
│       └── tests/
│
├── lib/                          # 공유 라이브러리들
│   ├── grpc-lib/
│   │   ├── pyproject.toml
│   │   ├── opendoor/            # 네임스페이스 패키지
│   │   │   └── grpc/
│   │   └── tests/
│   └── common-lib/
│       ├── pyproject.toml
│       ├── opendoor/
│       │   └── common/
│       └── tests/
│
└── tools/                        # 개발 도구
    ├── pippy/                    # 서비스 생성기
    ├── pyfmt/                    # 코드 포맷터
    └── ci/                       # CI/CD 인프라
```

**핵심:** 모든 라이브러리가 `opendoor.*` 네임스페이스 하위. 약 12개 내부 라이브러리, 24개 서비스 운영.

---

## 4. 패턴 비교 분석

| 관점 | 단일 패키지 (NegMAS) | 멀티 패키지 모노레포 |
|------|----------------------|---------------------|
| 규모 | 하나의 라이브러리/프레임워크 | 여러 서비스 + 공유 라이브러리 |
| import | `from negmas.situated import World` | `from myorg.shared import Model` |
| 의존성 | 단일 pyproject.toml | 패키지별 pyproject.toml |
| 배포 | PyPI에 하나로 배포 | 서비스별 Docker 이미지 |
| 적합한 경우 | 라이브러리/프레임워크 | 마이크로서비스 아키텍처 |

---

## 5. 핵심 질문에 대한 답

### Q: 게임/도메인 코드는 별도 패키지? 공유의 서브패키지?

**실제 사례에서 나타난 3가지 접근:**

1. **별도 저장소** (NegMAS + SCML 방식)
   - 프레임워크와 게임이 완전 분리
   - 게임이 프레임워크를 pip 의존성으로 참조
   - 장점: 깔끔한 분리, 독립 배포
   - 단점: 개발 중 양쪽 변경 시 불편

2. **같은 저장소, 별도 워크스페이스 멤버** (uv workspace 방식)
   - `libs/shared/`, `libs/game-economy/`, `apps/gamemaster/` 등
   - 장점: 한 커밋으로 공유코드+게임로직 동시 변경
   - 단점: 패키지 경계 설정 필요

3. **단일 패키지의 서브패키지** (NegMAS 내부 구조 방식)
   - `agentpit.shared.models`, `agentpit.game.economy`, `agentpit.web`
   - 장점: 가장 단순, import 직관적
   - 단점: 서비스별 독립 배포 어려움

### Q: AgentPit에 가장 적합한 구조는?

현재 AgentPit 상황:
- uv 이미 사용 중
- web, worker, gamemaster, evaluator 4개 서비스
- shared 모델 존재
- 서비스별 Docker 배포 예정

**추천: 패턴 A/B 하이브리드 (uv workspace)**

```
AgentPit/
├── pyproject.toml              # 루트: workspace 정의
├── uv.lock                     # 단일 lockfile
│
├── libs/                        # 공유 라이브러리들
│   ├── shared/                  # 모든 서비스가 쓰는 공통 코드
│   │   ├── pyproject.toml
│   │   └── src/agentpit/shared/
│   │       ├── models/          # Pydantic 모델 등
│   │       ├── data/            # 정적 데이터
│   │       └── config.py
│   │
│   └── game-engine/             # 게임/경제 시뮬레이션 로직
│       ├── pyproject.toml
│       └── src/agentpit/game/
│           ├── economy/         # 경제 엔진
│           ├── trading/         # 거래 로직
│           └── scenarios/       # 시나리오 정의
│
├── apps/                        # 배포 가능한 서비스들
│   ├── web/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml       # depends on: agentpit-shared
│   │   └── src/agentpit/web/
│   │
│   ├── worker/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml       # depends on: agentpit-shared, agentpit-game
│   │   └── src/agentpit/worker/
│   │
│   ├── gamemaster/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml       # depends on: agentpit-shared, agentpit-game
│   │   └── src/agentpit/gamemaster/
│   │
│   └── evaluator/
│       ├── Dockerfile
│       ├── pyproject.toml       # depends on: agentpit-shared
│       └── src/agentpit/evaluator/
│
├── docs/
└── dev-notes/
```

**이 구조의 장점:**
- 의존 방향이 명확: `apps → libs` (역방향 불가)
- 서비스별 Docker 이미지 빌드 시 필요한 libs만 포함
- `uv sync --package=agentpit-web` 같이 서비스별 개발 가능
- game-engine을 shared에서 분리해 도메인 경계 유지

---

## Sources

- [carderne/postmodern-mono](https://github.com/carderne/postmodern-mono) - uv workspace 모노레포 템플릿
- [mvoss02/uv_workspaces_example](https://github.com/mvoss02/uv_workspaces_example) - services + packages 구조
- [matanby/python-monorepo-template](https://github.com/matanby/python-monorepo-template) - 서비스/패키지 생성 스크립트 포함
- [tweag/python-monorepo-example](https://github.com/tweag/python-monorepo-example) - Pants + 네임스페이스 패키지
- [niqodea/python-monorepo](https://github.com/niqodea/python-monorepo) - 네임스페이스 기반 모노레포 템플릿
- [Tweag Blog: Python Monorepo Part 1](https://www.tweag.io/blog/2023-04-04-python-monorepo-1/) - 구조와 도구 설명
- [Opendoor Python Monorepo](https://medium.com/opendoor-labs/our-python-monorepo-d34028f2b6fa) - 프로덕션 모노레포 사례
- [NegMAS GitHub](https://github.com/yasserfarouk/negmas) - 멀티에이전트 협상 시뮬레이션
- [SCML GitHub](https://github.com/yasserfarouk/scml) - NegMAS 기반 게임
- [uv Workspaces 공식 문서](https://docs.astral.sh/uv/concepts/projects/workspaces/) - uv 워크스페이스 가이드
- [uv Monorepo Best Practices Issue](https://github.com/astral-sh/uv/issues/10960) - 커뮤니티 논의
- [Python Monorepo with UV (Medium)](https://medium.com/@life-is-short-so-enjoy-it/python-monorepo-with-uv-f4ced6f1f425)
- [Building a Python Monorepo with UV (2026)](https://medium.com/@naorcho/building-a-python-monorepo-with-uv-the-modern-way-to-manage-multi-package-projects-4cbcc56df1b4)
- [pydevtools: How to set up uv monorepo](https://pydevtools.com/handbook/how-to/how-to-set-up-a-python-monorepo-with-uv-workspaces/)
