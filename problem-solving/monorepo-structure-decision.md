# Python 모노레포 구조 선택

## 후보 패턴 4가지

| 패턴 | 구조 | 대표 사례 |
|------|------|----------|
| A. 단일 패키지 서브패키지 | `src/myapp/{web,worker,shared}/` | NegMAS, 소규모 프로젝트 |
| B. `services/` + `packages/` | 서비스/라이브러리 명시 분리 | mvoss02/uv_workspaces_example |
| C. `libs/` + `apps/` | 공유 라이브러리 + 서비스 분리 | Opendoor (12 lib, 24 서비스 프로덕션) |
| D. 멀티 레포 | 서비스별 별도 저장소 | 대규모 팀 (Netflix 일부) |

## 선택: C. `libs/` + `apps/`

```
AgentPit/
├── libs/
│   ├── shared/          ← 모든 게임 공통 (Session, 메시지)
│   └── economy-sim/     ← 게임별 코드 (Resource, Recipe, tools)
├── apps/
│   ├── web/
│   ├── worker/
│   └── gamemaster/
├── pyproject.toml       ← uv workspace 루트
└── uv.lock
```

## 선택 근거

1. **libs와 apps의 역할이 명확** — 라이브러리(import되는 것)와 서비스(실행되는 것) 분리
2. **게임 추가 시 확장 용이** — `libs/negotiation/` 추가만 하면 됨. shared 코드 수정 불필요
3. **서비스별 독립 배포** — 각 app이 자체 pyproject.toml + Dockerfile. Docker Compose → K8s 전환 시 코드 변경 없음
4. **uv workspace 지원** — `uv.lock` 하나로 전체 의존성 일관성 유지
5. **프로덕션 검증** — Opendoor가 24개 서비스를 이 구조로 운영 중

## 탈락 근거

- **A (단일 패키지)**: 게임별 코드와 공통 코드가 섞임. 게임 추가 시 shared가 비대해짐
- **B (services + packages)**: C와 유사하지만 네이밍이 덜 직관적
- **D (멀티 레포)**: 솔로 개발자가 여러 레포 관리는 오버헤드. shared 코드 동기화 어려움

## 레퍼런스

- [Opendoor Python Monorepo](https://medium.com/opendoor-labs/our-python-monorepo-d34028f2b6fa) — 프로덕션 사례
- [uv Workspaces 공식 문서](https://docs.astral.sh/uv/concepts/projects/workspaces/) — uv workspace 설정법
- [carderne/postmodern-mono](https://github.com/carderne/postmodern-mono) — uv workspace 예제
- [mvoss02/uv_workspaces_example](https://github.com/mvoss02/uv_workspaces_example) — services + packages 예제
- [NegMAS GitHub](https://github.com/yasserfarouk/negmas) — 게임 시뮬레이션 프레임워크 구조
- [SCML GitHub](https://github.com/yasserfarouk/scml) — 게임별 분리 사례
