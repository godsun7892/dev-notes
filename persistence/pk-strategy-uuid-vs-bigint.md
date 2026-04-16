# PK 전략: UUID v4 vs UUID v7 vs BIGINT

## PostgreSQL에서 UUID는 문자열이 아니다

UUID 타입은 내부적으로 16 bytes 고정 바이너리로 저장된다.
`'550e8400-e29b-41d4-a716-446655440000'` 문자열은 입출력 시에만 사용.

```
보이는 것: '550e8400-e29b-41d4-a716-446655440000'  (36 chars)
저장되는 것: 0x550e8400e29b41d4a716446655440000      (16 bytes)
비교 연산:  memcmp(16 bytes, 16 bytes)
```

## 성능을 결정하는 건 비교 비용이 아니라 page 접근 패턴

B-tree 인덱스에서 키 비교 자체(8 byte 정수 vs 16 byte memcmp)는 나노초 수준으로 무시 가능.
진짜 비용은 **INSERT 시 어느 leaf page에 접근하느냐**에서 온다.

### BIGINT (auto-increment): 순차 증가

항상 가장 오른쪽 leaf page에 INSERT. 해당 page가 항상 shared_buffers(메모리)에 있으므로 디스크 I/O 없음.

### UUID v7: 시간순 단조 증가

앞 48 bit가 timestamp. 시간순으로 증가하므로 BIGINT과 동일하게 오른쪽 leaf에만 INSERT.
같은 ms 안에서도 sub-millisecond fraction으로 단조 증가 보장.

### UUID v4: 완전 랜덤

매 INSERT마다 랜덤한 leaf page에 접근. 해당 page가 메모리에 없으면 디스크에서 읽어와야 함.
테이블이 커질수록 인덱스가 shared_buffers를 초과 → random disk I/O 폭증.

## 벤치마크 요약

### INSERT 성능 (ardentperf, PG 17 beta, 21M 행 + 1M 동시 INSERT)

| 방식 | TPS | INSERT 시간 |
|------|-----|------------|
| BIGINT | 3,480 | 290s |
| UUID v7 | 3,420 | 290s |
| UUID v4 | 2,670 | 375s |

UUID v7은 BIGINT의 98.3% (1.7% 차이, 측정 오차 수준). UUID v4는 29% 느림.

출처: Kyle Hailey — UUID Benchmark War (ardentperf.com)

### UPDATE 성능 (Cybertec, 10M 행 + 1M UPDATE)

| PK 타입 | 인덱스 스캔 buffer hits |
|---------|----------------------|
| BIGINT | 27,332 |
| UUID v4 | 8,562,960 (313배) |

UUID v4는 UPDATE에서 BIGINT 대비 313배 buffer hits. v7은 BIGINT과 유사.

출처: Cybertec — Unexpected downsides of UUID keys

### Leaf page fill density (1M UPDATE 후)

| PK 타입 | Leaf fill |
|---------|-----------|
| BIGINT | 97.64% |
| UUID v7 | 90.09% |
| UUID v4 | 79.06% |

출처: Cybertec (Andy Atkinson 재인용)

### 대규모 조회 (Medium, 75M 행, PG 15, AWS RDS)

| 연산 | UUID v7 | BIGSERIAL |
|------|---------|-----------|
| 단건 조회 | 12.4ms | 0.8ms |
| JOIN (1:N) | 84ms | 3.2ms |
| Range 쿼리 | 18.7ms | 1.2ms |
| FK 인덱스 크기 | 847MB | 423MB |

75M 행에서 BIGSERIAL이 조회 15~26배 빠름. 단, 이 규모는 대부분의 프로젝트가 도달하지 않음.

출처: jamauriceholt — UUID v7 vs BIGSERIAL (Medium)

### 인덱스 크기

| PK 타입 | 인덱스 크기 (동일 행수 기준) |
|---------|--------------------------|
| BIGINT (8 byte) | 100% (기준) |
| UUID v7 | +25% |
| UUID v4 | +35% (page split bloat 포함) |

키 크기만 보면 16/8 = 2배여야 하지만, B-tree tuple 헤더 오버헤드로 실측 1.25~1.35배.

### 변곡점

| 규모 | 차이 |
|------|------|
| < 100K 행 | 무시 가능 |
| 100K ~ 1M | 측정 가능하지만 병목은 아님 |
| 1M ~ 10M | TEXT UUID 테이블 54% 커짐, 인덱스 85% 커짐 |
| 10M+ | v4는 심각. v7도 BIGINT 대비 읽기 느려짐 |

핵심: 변곡점은 행수가 아니라 **인덱스 크기 vs shared_buffers 비율**이 결정.
인덱스가 shared_buffers 안에 들어오면 v7 vs BIGINT 차이는 측정 불가.

## 저장 크기 비교

| 타입 | 바이트 |
|------|--------|
| INTEGER | 4 |
| BIGINT | 8 |
| UUID (native) | 16 |
| TEXT UUID (문자열) | 37~40 |

## 실제 프로젝트 PK 전략

| 프로젝트 | PK 전략 |
|----------|---------|
| cosmicpython (aggregate root) | 도메인 자연키(String) as PK |
| cosmicpython (자식 엔티티) | surrogate Integer |
| Vernon IDDD_Samples (DDD 교과서) | 도메인 UUID를 저장 키로 직접 사용 |
| Kgrzybek modular-monolith (C#) | 도메인 UUID = DB PK |

DDD 순수주의: 도메인 UUID as PK. 실용주의: surrogate integer 혼용.

업계 대규모 서비스에서는 Dual ID 패턴(BIGINT PK 내부 + UUID 외부) 사용.

## 결론

- UUID v4는 쓰면 안 됨 (랜덤 page 접근으로 대규모에서 치명적)
- UUID v7 ≈ BIGINT (시간순 단조 증가로 동일 접근 패턴)
- 수십만~수백만 행 규모에서 v7 vs BIGINT 차이를 체감하기 어려움
- 10M+ 행에서 JOIN 성능은 BIGINT이 우위
- PostgreSQL 18 (2025-09)부터 native uuidv7() 함수 제공
