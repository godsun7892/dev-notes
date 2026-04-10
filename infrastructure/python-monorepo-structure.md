# Python 모노레포 구조 — 학습 노트

| 항목 | 내용 |
|------|------|
| 핵심 질문 | 여러 서비스가 공유 코드를 사용하는 Python 프로젝트를 어떻게 구조화하는가? |
| 도구 | uv workspace, Pants, Bazel, Poetry workspaces |

---

## 1. 후보 패턴 4가지

| 패턴 | 구조 | 대표 사례 | 적합한 경우 |
|------|------|----------|-----------|
| **A. 단일 패키지 + 서브패키지** | `src/myapp/{web,worker,shared}/` | NegMAS, 소규모 라이브러리 | 단일 라이브러리/프레임워크 |
| **B. `services/` + `packages/`** | 서비스/라이브러리 명시 분리 | mvoss02/uv_workspaces_example | 명시적 네이밍 선호 |
| **C. `libs/` + `apps/`** | 공유 라이브러리 + 서비스 분리 | Opendoor (12 lib, 24 서비스) | 마이크로서비스 + 공유 코드 |
| **D. 멀티 레포** | 서비스별 별도 저장소 | 일부 대규모 팀 | 다른 팀/언어/배포 주기 |

---

## 2. 패턴 A — 단일 패키지 (NegMAS 방식)

**단일 저장소, 도메인별 서브패키지로만 분리.**

```
negmas/
└── src/
    └── negmas/                  # 하나의 큰 패키지
        ├── __init__.py
        │
        │  # === 핵심 도메인 모듈 ===
        ├── mechanisms.py        # 협상 프로토콜
        ├── protocols.py         # 프로토콜 인터페이스
        │
        │  # === 도메인 서브패키지 ===
        ├── negotiators/         # 협상 에이전트
        │   ├── negotiator.py
        │   ├── controller.py
        │   └── components/      # 더 깊은 하위 분류
        │
        ├── situated/            # 시뮬레이션 월드
        │   ├── world.py
        │   ├── agent.py
        │   ├── contract.py
        │   └── awi.py
        │
        ├── outcomes/            # 협상 결과
        ├── preferences/         # 효용 함수
        ├── scenarios/           # 시나리오
        ├── tournaments/         # 토너먼트
        │
        │  # === 인프라 ===
        ├── helpers/
        ├── config.py
        ├── events.py
        ├── exceptions.py
        └── tests/
```

### 특징
- 단일 `pyproject.toml`
- PyPI에 단일 패키지로 배포
- import: `from negmas.situated import World`

### 장점
- 가장 단순
- import 직관적
- 패키지 경계 설정 불필요

### 단점
- 서비스별 독립 배포 어려움
- 코드 늘어나면 도메인 경계 흐려짐
- 라이브러리 vs 실행 가능한 서비스 구분 모호

---

## 3. 패턴 B — `services/` + `packages/` (mvoss02 방식)

**서비스와 라이브러리를 명시적으로 분리.**

```
project/
├── pyproject.toml
├── uv.lock
├── packages/                        # 공유 라이브러리
│   ├── my_package_1/
│   │   ├── pyproject.toml
│   │   └── src/
│   │       └── my_package_1/
│   └── my_package_2/
│       ├── pyproject.toml
│       └── src/
│           └── my_package_2/
└── services/                        # 배포 가능한 서비스
    ├── data_ingestion/
    │   ├── pyproject.toml
    │   └── data_ingest/
    └── data_preprocessing/
        ├── pyproject.toml
        └── data_pre/
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
```

---

## 4. 패턴 C — `libs/` + `apps/` (Opendoor 방식)

**가장 확장성 좋은 패턴. uv workspace 친화적.**

```
project/
├── pyproject.toml              # 루트: workspace 정의만
├── uv.lock                     # 단일 lockfile (전체 공유)
│
├── libs/                       # 공유 라이브러리
│   ├── shared/
│   │   ├── pyproject.toml
│   │   └── src/myorg/shared/   # 네임스페이스
│   │       ├── models/
│   │       └── config.py
│   │
│   └── domain-engine/
│       ├── pyproject.toml
│       └── src/myorg/domain/
│
├── apps/                       # 배포 가능한 서비스
│   ├── web/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml      # depends on: myorg-shared
│   │   └── src/myorg/web/
│   │
│   ├── worker/
│   │   ├── Dockerfile
│   │   ├── pyproject.toml
│   │   └── src/myorg/worker/
│   │
│   └── api/
│       ├── Dockerfile
│       ├── pyproject.toml
│       └── src/myorg/api/
│
└── tools/                      # 개발 도구 (선택)
```

루트 `pyproject.toml`:
```toml
[tool.uv.workspace]
members = ["libs/*", "apps/*"]   # glob으로 자동 인식
```

서비스의 `pyproject.toml`:
```toml
[project]
dependencies = ["myorg-shared"]      # 공유 라이브러리 의존

[tool.uv.sources]
myorg-shared = { workspace = true }  # PyPI 아닌 워크스페이스에서 해결
```

### 장점
- `libs`(import되는 것)와 `apps`(실행되는 것)의 역할이 명확
- 의존 방향이 명확: `apps → libs` (역방향 불가)
- 서비스별 독립 Docker 이미지 빌드, 필요한 libs만 포함
- `uv sync --package=myorg-web` 같이 서비스별 개발 가능
- 새 라이브러리/서비스 추가 시 디렉토리만 추가
- Opendoor가 24개 서비스를 이 구조로 프로덕션 운영 중

### 단점
- 패키지 경계를 미리 설계해야 함
- 작은 프로젝트엔 오버킬

---

## 5. 패턴 D — 별도 저장소 + 외부 의존성 (NegMAS + SCML 방식)

**프레임워크와 사용자 코드를 완전 분리.**

```
negmas/                          # 별도 저장소 (프레임워크)
└── src/negmas/...

scml/                            # 별도 저장소 (게임)
└── src/
    └── scml/
        ├── common.py
        ├── runner.py
        ├── oneshot/             # OneShot 게임 모드
        ├── std/                 # 표준 게임 모드
        ├── scml2019/            # 연도별 버전
        └── scml2020/
```

`scml`이 `negmas`를 `pyproject.toml`의 dependency로 참조.

### 장점
- 깔끔한 분리, 독립 배포 주기
- 서로 다른 팀이 소유 가능
- 라이선스/공개 범위 다르게 가능 (예: 프레임워크는 오픈소스, 사용자 코드는 비공개)

### 단점
- 개발 중 양쪽 변경 시 publish-install 사이클 필요
- 버전 skew 위험 (양쪽 호환성 직접 관리)
- 1인 개발자에겐 오버헤드

---

## 6. uv workspace 메커니즘

### 워크스페이스 정의

루트 `pyproject.toml`:
```toml
[project]
name = "myorg"
version = "0.1.0"
requires-python = ">=3.12"

[tool.uv.workspace]
members = ["libs/*", "apps/*"]

[tool.uv.sources]
myorg-shared = { workspace = true }
myorg-domain = { workspace = true }
```

`members`에 glob 패턴 사용 가능 (`libs/*`, `apps/*`). 새 패키지 추가 시 디렉토리만 만들면 자동 인식.

### 워크스페이스 의존성

서비스의 `pyproject.toml`:
```toml
[project]
name = "myorg-web"
dependencies = [
    "myorg-shared",
    "fastapi>=0.110",
]

[tool.uv.sources]
myorg-shared = { workspace = true }
```

`workspace = true`는 "PyPI에서 받지 말고 같은 워크스페이스에서 해결"의 의미. 로컬 코드 변경이 즉시 반영됨.

### 단일 lockfile

전체 워크스페이스가 하나의 `uv.lock` 공유. 모든 서비스가 같은 패키지 버전 사용.

### 서비스별 동기화

```bash
uv sync --package=myorg-web    # web만 설치
uv sync                         # 전체 설치
```

Docker 빌드 시 필요한 패키지만 포함 가능 (이미지 크기 절감).

---

## 7. 산업계 사례 — Opendoor

```
opendoor-monorepo/
├── projects/                    # 배포 가능한 서비스들
│   └── my-service/
│       ├── Dockerfile
│       ├── pyproject.toml
│       ├── my_service/
│       └── tests/
│
├── lib/                         # 공유 라이브러리들
│   ├── grpc-lib/
│   │   ├── pyproject.toml
│   │   ├── opendoor/            # 네임스페이스
│   │   │   └── grpc/
│   │   └── tests/
│   └── common-lib/
│       ├── pyproject.toml
│       ├── opendoor/
│       │   └── common/
│       └── tests/
│
└── tools/                       # 개발 도구
    ├── pippy/                   # 서비스 생성기
    ├── pyfmt/                   # 코드 포맷터
    └── ci/                      # CI/CD 인프라
```

- 모든 라이브러리가 `opendoor.*` 네임스페이스 하위
- 약 12개 내부 라이브러리, 24개 서비스 운영
- 사실상 패턴 C와 동일 (`projects/` = `apps/`, `lib/` = `libs/`)

---

## 8. 패턴 비교 분석

| 관점 | 단일 패키지 (A) | 멀티 패키지 모노레포 (B/C) | 멀티 레포 (D) |
|------|---------------|---------------------------|--------------|
| 규모 | 단일 라이브러리/프레임워크 | 마이크로서비스 + 공유 코드 | 다른 팀/언어 |
| import | `from project.x import Y` | `from myorg.shared import Y` | `import external_lib` |
| 의존성 관리 | 단일 pyproject | 패키지별 pyproject + 단일 lockfile | 패키지별 pyproject + 별도 lockfile |
| 배포 | PyPI 단일 패키지 | 서비스별 Docker | 저장소별 독립 |
| 코드 변경 | 단일 PR | 단일 PR (atomic) | 여러 PR + 순서 관리 |
| 서비스 추가 | 서브패키지 추가 | 디렉토리 추가 | 새 저장소 |

---

## 9. 패턴 선택 기준

### A (단일 패키지)
- 라이브러리/프레임워크 하나만 만드는 경우
- 서비스를 여러 개 배포할 계획 없음
- 빠른 시작, 최소 설정 원함

### B/C (모노레포)
- 여러 서비스가 공유 코드 사용
- 서비스별 독립 Docker 배포 필요
- 단일 PR로 schema 변경 + 사용처 동시 수정 원함
- C는 명시적 `libs`/`apps` 분리 (Opendoor 검증), B는 더 자유로운 네이밍

### D (멀티 레포)
- 다른 팀이 소유
- 다른 언어/런타임
- 다른 배포 주기 (예: 라이브러리 안정, 사용자 코드 빠르게 변동)
- 외부 공개 (오픈소스) vs 비공개 라이선스 분리
- 보안 격리 (다른 access 권한)

---

## 10. 흔한 함정

### 10.1 너무 일찍 분리
초기에 패턴 D 또는 너무 많은 libs/apps로 쪼개면, schema 변경 시 여러 패키지/저장소를 동시에 수정하는 비용이 발생. **시작은 최소 분리, 필요할 때 쪼개기**가 안전.

### 10.2 네임스페이스 누락
`libs/shared/src/shared/` 같이 네임스페이스 없이 만들면 다른 라이브러리와 import 충돌 가능. `libs/shared/src/myorg/shared/`처럼 조직 네임스페이스 필요.

### 10.3 lockfile 분산
패키지마다 별도 lockfile을 두면 같은 의존성의 버전이 달라져서 디버깅 지옥. uv workspace는 단일 lockfile 강제 — 이걸 활용해야 함.

### 10.4 apps에서 다른 apps import
`apps/worker`가 `apps/web`을 import하는 순간 의존 그래프가 깨짐. **`apps → libs`만 허용, `apps → apps` 금지.** 공통 코드는 무조건 `libs`로 빼야 함.

### 10.5 libs가 너무 비대해짐
모든 공통 코드를 `libs/shared`에 다 넣다 보면 한 라이브러리가 비대해짐. 도메인별로 분리: `libs/shared`(범용), `libs/domain-A`(특정 도메인), `libs/domain-B` 등.

---

## 11. 결론

여러 서비스 + 공유 코드 패턴이면 **C (`libs/` + `apps/`)** 가 가장 검증되고 확장성 좋다. uv workspace + 단일 lockfile + glob members 조합이 표준.

작은 프로젝트는 **A (단일 패키지)**로 시작했다가 서비스가 늘어나면 C로 마이그레이션 가능.

**D (멀티 레포)** 는 다른 팀 소유, 다른 언어, 다른 배포 주기 같은 명확한 이유가 있을 때만.

---

## References

### 모노레포 템플릿
- [carderne/postmodern-mono](https://github.com/carderne/postmodern-mono) — uv workspace 모노레포 템플릿
- [mvoss02/uv_workspaces_example](https://github.com/mvoss02/uv_workspaces_example) — services + packages 구조
- [matanby/python-monorepo-template](https://github.com/matanby/python-monorepo-template) — 서비스/패키지 생성 스크립트 포함
- [tweag/python-monorepo-example](https://github.com/tweag/python-monorepo-example) — Pants + 네임스페이스 패키지
- [niqodea/python-monorepo](https://github.com/niqodea/python-monorepo) — 네임스페이스 기반 모노레포

### 산업계 사례
- [Opendoor Python Monorepo](https://medium.com/opendoor-labs/our-python-monorepo-d34028f2b6fa) — 12 lib, 24 서비스 프로덕션 운영
- [NegMAS GitHub](https://github.com/yasserfarouk/negmas) — 멀티에이전트 협상 시뮬레이션 (단일 패키지 사례)
- [SCML GitHub](https://github.com/yasserfarouk/scml) — NegMAS 기반 게임 (별도 레포 사례)

### 도구 문서
- [uv Workspaces 공식 문서](https://docs.astral.sh/uv/concepts/projects/workspaces/)
- [uv Monorepo Best Practices Issue](https://github.com/astral-sh/uv/issues/10960) — 커뮤니티 논의

### 가이드 글
- [Tweag Blog: Python Monorepo Part 1](https://www.tweag.io/blog/2023-04-04-python-monorepo-1/)
- [Python Monorepo with UV (Medium)](https://medium.com/@life-is-short-so-enjoy-it/python-monorepo-with-uv-f4ced6f1f425)
- [Building a Python Monorepo with UV (2026)](https://medium.com/@naorcho/building-a-python-monorepo-with-uv-the-modern-way-to-manage-multi-package-projects-4cbcc56df1b4)
- [pydevtools: How to set up uv monorepo](https://pydevtools.com/handbook/how-to/how-to-set-up-a-python-monorepo-with-uv-workspaces/)
