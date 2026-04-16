# 지역 / 시간 / 언어 처리 원칙 — 학습 노트

| 항목 | 내용 |
|------|------|
| Date | 2026-04-11 |
| 핵심 질문 | 셋을 어떻게 분리하고, 어디서 변환하며, 무엇을 저장하는가? |

---

## 1. 정석 패턴 개요 — 3축 분리

웹/모바일 백엔드 설계의 표준은 **`region` / `timezone` / `locale` 세 축을 독립적으로** 다루는 것이다. 셋은 사용자마다 자유롭게 조합되며 한 컬럼으로 묶을 수 없다.

| 축 | 의미 | 표준 | 예시 | 누가 결정 |
|---|------|------|------|----------|
| **Region (Country)** | 데이터/서버가 어디 있는가, 데이터 주권 | ISO 3166-1 alpha-2 | `KR`, `US`, `JP` | 인프라 + 법률 |
| **Timezone** | 시간을 어떻게 표시할까 | IANA tz database | `Asia/Seoul`, `America/Los_Angeles` | 사용자 거주지 |
| **Locale** | 어떤 언어/형식으로 표시할까 | BCP 47 | `ko-KR`, `en-US` | 사용자 선호 |

### 사용자 모델의 정석

```python
class User:
    user_id: UUID
    country_code: str  # ISO 3166: 'KR'
    timezone: str      # IANA: 'Asia/Seoul'
    locale: str        # BCP 47: 'ko-KR'
```

세 컬럼을 따로 두는 것이 표준이다. Django의 `django-timezone-field`, Rails의 `ActiveSupport::TimeZone`, Java의 `java.time.ZoneId` 등 모든 메이저 프레임워크가 같은 방식이다.

### 변환의 정석 — Functional Core / Imperative Shell

도메인 코드는 UTC만 다루고, 변환은 가장자리(직렬화, 표시)에서만 한다.

```
[사용자 입력] → 파싱 → UTC + locale 컨텍스트
                          ↓
                    [도메인 로직] (UTC만)
                          ↓
                    [DB] (UTC 저장)
                          ↓
[직렬화] ← UTC + 사용자 timezone/locale 적용 → [사용자 표시]
```

이 패턴은 Django, Rails, ASP.NET, Spring 등 모든 메이저 프레임워크의 디폴트 권장 사항이다.

---

## 2. 시간 — UTC + IANA + edge 변환

### 2.1 왜 UTC인가

시간대는 정치적이다. 매년 IANA tz database가 갱신된다 (정부가 timezone을 바꾸는 케이스):
- 러시아 2014: 모스크바 timezone 변경
- 베네수엘라 2016: UTC-4:30 → UTC-4:00
- 사모아 2011: 날짜선 이동 (12월 30일이 사라짐)
- 한국 1988~1989: 잠깐 서머타임 도입

UTC는 정치적이지 않고 천문학적으로 고정된 점이다. **DB에 UTC로 저장하면 정책이 어떻게 바뀌어도 데이터는 안 깨진다.**

### 2.2 PostgreSQL — `TIMESTAMP` vs `TIMESTAMPTZ`

```sql
-- ❌ Bad
created_at TIMESTAMP                    -- timezone 정보 없음
created_at TEXT                         -- 정렬/연산 깨짐

-- ✅ Good
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

`TIMESTAMP WITH TIME ZONE` (= `TIMESTAMPTZ`)은 이름과 달리 **내부적으로 UTC로 저장**한다. 입력값이 어느 timezone이든 UTC로 변환해 저장하고, 출력 시 세션 timezone으로 변환해 보여준다.

```sql
SET TIME ZONE 'Asia/Seoul';
INSERT INTO events (occurred_at) VALUES ('2026-04-11 15:00:00');
-- 내부 저장: '2026-04-11 06:00:00 UTC'

SET TIME ZONE 'America/Los_Angeles';
SELECT occurred_at FROM events;
-- 출력: '2026-04-10 23:00:00-07'
```

데이터는 동일, 보여주는 형태만 다름.

### 2.3 SQLAlchemy — `DateTime(timezone=True)`

```python
from sqlalchemy import Column, DateTime
from sqlalchemy.sql import func

class Event(Base):
    __tablename__ = "events"
    id = Column(Integer, primary_key=True)
    # ✅ timezone-aware
    occurred_at = Column(DateTime(timezone=True), nullable=False, server_default=func.now())
```

`timezone=True`를 빠뜨리면 PostgreSQL에서 `TIMESTAMP WITHOUT TIME ZONE`이 되어 timezone 정보가 손실된다.

### 2.4 Django — `USE_TZ = True`

```python
# settings.py
USE_TZ = True
TIME_ZONE = 'UTC'  # 백엔드 기본은 UTC
```

`USE_TZ = True`로 두면 Django ORM이 모든 datetime을 timezone-aware UTC로 다룬다. `datetime.datetime.now()` 대신 `django.utils.timezone.now()`를 써야 aware UTC가 나온다.

```python
from django.utils import timezone

now = timezone.now()  # aware UTC
```

### 2.5 Python — `datetime`의 함정

Python `datetime`은 두 종류가 있다:
- **naive**: timezone 정보 없음 (`datetime(2026, 4, 11, 15, 0)`)
- **aware**: timezone 정보 있음 (`datetime(2026, 4, 11, 15, 0, tzinfo=ZoneInfo("Asia/Seoul"))`)

**naive datetime은 거의 항상 버그의 원인**이다. 도메인 코드는 무조건 aware UTC를 쓴다.

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo  # Python 3.9+ stdlib

# ❌ Bad — naive
now = datetime.now()
# 이게 KST인지 UTC인지 모름. OS의 local timezone에 의존

# ❌ Bad — naive utcnow (Python 3.12+에서 deprecated)
now = datetime.utcnow()
# UTC지만 tzinfo가 None이라 naive. 다른 aware datetime과 비교/연산 시 TypeError

# ✅ Good — aware UTC
now = datetime.now(timezone.utc)
# tzinfo=UTC 명시. 어디서 실행해도 동일

# ✅ 표시 시점에만 변환
seoul = now.astimezone(ZoneInfo("Asia/Seoul"))
```

### 2.6 변환은 edge에서만

도메인 코드 안에서 `astimezone()`을 호출하는 건 안티패턴. 변환은 **표시(serialize) 직전에만**:

```python
# ❌ Bad — 도메인이 timezone을 알고 있음
def is_market_open(user_tz: ZoneInfo) -> bool:
    local_time = datetime.now(user_tz)
    return 9 <= local_time.hour < 18

# ✅ Good — 도메인은 UTC만, 정책도 UTC 기준
MARKET_OPEN_UTC = time(0, 0)   # 09:00 KST = 00:00 UTC
MARKET_CLOSE_UTC = time(9, 0)  # 18:00 KST = 09:00 UTC

def is_market_open(now_utc: datetime) -> bool:
    return MARKET_OPEN_UTC <= now_utc.time() < MARKET_CLOSE_UTC
```

### 2.7 UTC offset(`+09:00`)을 절대 저장하지 마라

```python
# ❌ Bad — DST 못 따라감
user.timezone = "+09:00"

# ✅ Good — IANA 이름
user.timezone = "Asia/Seoul"
```

`+09:00`은 "지금 이 순간"의 offset일 뿐. DST 적용 국가는 1년에 두 번 offset이 바뀌고, 정책 변경으로도 바뀐다. IANA 이름은 이 모든 변동을 자동으로 따라간다.

```python
from datetime import datetime
from zoneinfo import ZoneInfo

la = ZoneInfo("America/Los_Angeles")
summer = datetime(2026, 7, 1, 12, 0, tzinfo=la)
winter = datetime(2026, 12, 1, 12, 0, tzinfo=la)

print(summer.utcoffset())  # -7:00:00 (PDT, 서머타임)
print(winter.utcoffset())  # -8:00:00 (PST, 표준시)
# 같은 IANA 이름인데 offset 자동 변동
```

### 2.8 미래 시각의 함정 — wall clock + tz 별도 저장

이건 굉장히 미묘한 함정이다. 미래 시각(예약, 일정)을 UTC로 변환해 저장하면 안 된다.

**예시**: "2027년 1월 1일 09:00 KST에 회의 시작"

```python
# ❌ Bad
meeting_kst = datetime(2027, 1, 1, 9, 0, tzinfo=ZoneInfo("Asia/Seoul"))
meeting_utc = meeting_kst.astimezone(timezone.utc)
db.save(meeting_utc)  # 2026-12-31 24:00 UTC
```

문제: **만약 한국이 그때까지 timezone 정책을 바꾸면** (예: 2026년 9월에 "한국 표준시 = UTC+9:30"으로 변경), 저장된 UTC 시각은 9시가 아닌 다른 시각이 된다.

**해결**: wall clock + timezone을 별도로 저장.

```sql
CREATE TABLE meetings (
    meeting_id       UUID PRIMARY KEY,
    scheduled_local  TIMESTAMP NOT NULL,    -- '2027-01-01 09:00:00' (naive)
    scheduled_tz     TEXT NOT NULL          -- 'Asia/Seoul'
);
```

조회 시점에 IANA 룩업해서 정확한 UTC 계산:

```python
def get_actual_start_utc(scheduled_local: datetime, scheduled_tz: str) -> datetime:
    """현재 IANA tz database 기준으로 UTC 계산.
    그 사이에 정책이 바뀌었어도 의도된 wall clock 시각이 유지됨."""
    tz = ZoneInfo(scheduled_tz)
    aware_local = scheduled_local.replace(tzinfo=tz)
    return aware_local.astimezone(timezone.utc)
```

이 패턴은 Google Calendar, Outlook 등 메이저 캘린더 서비스가 모두 사용한다.

---

## 3. 언어 (Locale)

### 3.1 BCP 47 표준

언어 + 지역 조합. `language[-region]`:

| 코드 | 의미 |
|------|------|
| `ko` | 한국어 (지역 무관) |
| `ko-KR` | 한국어 (한국) |
| `en` | 영어 |
| `en-US` | 미국 영어 |
| `en-GB` | 영국 영어 (color → colour) |
| `zh-Hans` | 중국어 간체 |
| `zh-Hant` | 중국어 번체 |
| `pt-BR` | 브라질 포르투갈어 |

### 3.2 Locale 감지 우선순위 (정석)

```python
def resolve_locale(user, request) -> str:
    # 1순위: 사용자 명시 설정
    if user.locale:
        return user.locale

    # 2순위: HTTP Accept-Language 헤더
    accept = request.headers.get("Accept-Language")
    if accept:
        # 'ko-KR,ko;q=0.9,en-US;q=0.8,en;q=0.7'
        parsed = parse_accept_language(accept)
        for lang, _ in parsed:
            if lang in SUPPORTED_LOCALES:
                return lang

    # 3순위: 기본값
    return "en-US"
```

이 우선순위는 W3C i18n 권장 사항이다.

### 3.3 메시지 + 변수 분리 (ICU MessageFormat)

가장 흔한 안티패턴: 변수가 번역 메시지에 박힌다.

```python
# ❌ Bad — 복수형/조사 처리 망가짐
{
    "ko": "5달러 부족",
    "en": "Need 5 dollars more"
}
# 변수가 1이 되면? 영어는 "1 dollars" (s 떼야 함, 안 뗌)
```

ICU MessageFormat은 변수 + 복수형 + 성별을 표준으로 처리:

```json
{
  "insufficient_cash": {
    "en": "Need {amount, plural, one {# dollar} other {# dollars}} more",
    "ko": "{amount}달러 부족"
  }
}
```

```python
from icu import MessageFormat, Locale

msg = MessageFormat(
    "Need {amount, plural, one {# dollar} other {# dollars}} more",
    Locale("en_US")
)
print(msg.format({"amount": 1}))   # "Need 1 dollar more"
print(msg.format({"amount": 5}))   # "Need 5 dollars more"
```

ICU MessageFormat은 Mozilla, Google, Apple, Microsoft가 모두 사용하는 사실상의 표준이다.

### 3.4 Python 라이브러리

| 용도 | 라이브러리 |
|------|----------|
| 메시지 번역 (gettext) | `gettext` (stdlib), `babel` |
| ICU MessageFormat | `PyICU`, `icu-messageformat-parser` |
| 숫자/날짜 형식 | `babel.numbers`, `babel.dates` |
| 통화 | `babel.numbers.format_currency` |

```python
from babel.dates import format_datetime
from babel.numbers import format_currency
from datetime import datetime, timezone

dt = datetime.now(timezone.utc)
print(format_datetime(dt, locale='ko_KR'))
# '2026. 4. 11. 오후 6:00:00'

print(format_currency(1234.56, 'USD', locale='ko_KR'))
# 'US$1,234.56'

print(format_currency(1234.56, 'USD', locale='en_US'))
# '$1,234.56'

print(format_currency(1234.56, 'KRW', locale='ko_KR'))
# '₩1,235' (원화는 소수점 없음, 자동 반올림)
```

---

## 4. 지역 (Region)

### 4.1 ISO 3166-1 alpha-2

2글자 국가 코드. `KR`, `US`, `JP`, `CN`, `DE` ...

`country_code`로 저장하면 데이터 주권 + 통계 + 라우팅에 다 쓸 수 있다.

### 4.2 데이터 주권

법적으로 사용자 데이터를 사용자 거주국에 저장해야 할 수 있다:

| 법 | 요구사항 |
|----|---------|
| GDPR (EU) | EU 사용자 데이터는 EU region 우선 |
| 한국 개인정보보호법 | 일정 정보는 국내 보관 |
| 중국 사이버보안법 | 중국 사용자 데이터 중국 내 저장 |
| 러시아 데이터 주권법 | 러시아 사용자 데이터 러시아 내 저장 |

→ `users.country_code`로 사용자를 적절한 region에 라우팅. 한 region에서 다른 region으로 데이터 이동 시 법적 검토 필요.

### 4.3 AWS multi-region 라우팅

```
사용자 (country_code='KR')
  → Route 53 Geolocation Routing
  → ap-northeast-2 (Seoul) cluster

사용자 (country_code='US', timezone='America/Los_Angeles', locale='ko-KR')
  → Route 53 Geolocation Routing
  → us-west-2 (Oregon) cluster
  → 데이터: us-west-2 RDS
  → UI: 한국어 (locale 기반)
  → 시간: LA 시간 (timezone 기반)
```

ISO 3166 country code ↔ AWS region은 1:1이 아니다. 매핑 테이블 필요:

```python
COUNTRY_TO_REGION = {
    "KR": "ap-northeast-2",
    "JP": "ap-northeast-1",
    "US": "us-west-2",
    "DE": "eu-central-1",
    "GB": "eu-west-2",
    # ...
}
```

### 4.4 글로벌 데이터

리더보드, 검색 인덱스 같은 region 무관 데이터는 별도:
- DynamoDB Global Tables
- Aurora Global Database
- 단일 region에 모아 처리하는 글로벌 워커

---

## 5. 권장 스키마 (일반)

```sql
-- 사용자 (3축 분리)
CREATE TABLE users (
    user_id       UUID PRIMARY KEY,
    email         TEXT UNIQUE NOT NULL,
    country_code  CHAR(2) NOT NULL,        -- ISO 3166: 'KR'
    timezone      TEXT NOT NULL,           -- IANA: 'Asia/Seoul'
    locale        TEXT NOT NULL,           -- BCP 47: 'ko-KR'
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 이미 발생한 이벤트 (UTC)
CREATE TABLE events (
    event_id      UUID PRIMARY KEY,
    occurred_at   TIMESTAMPTZ NOT NULL,    -- 무조건 UTC
    region        CHAR(2)                  -- 발생 region (집계용)
);

-- 미래 예약 (wall clock + tz)
CREATE TABLE scheduled_meetings (
    meeting_id       UUID PRIMARY KEY,
    scheduled_local  TIMESTAMP NOT NULL,    -- naive wall clock
    scheduled_tz     TEXT NOT NULL          -- 'Asia/Seoul'
);
```

---

## 6. 코드 사례 — 오픈소스 레퍼런스

### 6.1 Django 프로젝트의 timezone 처리

```python
# settings.py
USE_TZ = True
TIME_ZONE = 'UTC'
LANGUAGE_CODE = 'en-us'

LANGUAGES = [
    ('en', 'English'),
    ('ko', 'Korean'),
]

USE_I18N = True
USE_L10N = True

# models.py
from django.db import models
from django.utils.translation import gettext_lazy as _

class User(models.Model):
    country_code = models.CharField(max_length=2)  # ISO 3166
    timezone = models.CharField(max_length=64, default='UTC')  # IANA
    locale = models.CharField(max_length=10, default='en-US')  # BCP 47

# views.py
from django.utils import timezone

def some_view(request):
    now = timezone.now()  # aware UTC, 항상
    ...

# templates/event.html
{% load tz %}
{% load i18n %}
{% timezone user.timezone %}
    {{ event.occurred_at }}
{% endtimezone %}
{% trans "Welcome" %}
```

### 6.2 FastAPI + SQLAlchemy

```python
from datetime import datetime, timezone
from sqlalchemy import Column, DateTime, String
from sqlalchemy.sql import func
from pydantic import BaseModel
from zoneinfo import ZoneInfo
from babel.dates import format_datetime

# Model
class EventORM(Base):
    __tablename__ = "events"
    id = Column(Integer, primary_key=True)
    occurred_at = Column(DateTime(timezone=True), nullable=False, server_default=func.now())

# Pydantic schema
class EventOut(BaseModel):
    id: int
    occurred_at: datetime         # ISO 8601 + offset
    occurred_at_display: str      # 사용자 친화 문자열

# Endpoint
@app.get("/events/{event_id}")
def get_event(event_id: int, current_user=Depends(get_current_user)):
    event = db.get(EventORM, event_id)
    user_tz = ZoneInfo(current_user.timezone)
    user_locale = current_user.locale.replace("-", "_")
    return EventOut(
        id=event.id,
        occurred_at=event.occurred_at,
        occurred_at_display=format_datetime(
            event.occurred_at.astimezone(user_tz),
            locale=user_locale,
        ),
    )
```

### 6.3 미래 시각 (캘린더 패턴)

Google Calendar, iCalendar (RFC 5545)가 사용하는 패턴:

```python
from datetime import datetime
from zoneinfo import ZoneInfo, ZoneInfoNotFoundError

class Meeting(BaseModel):
    title: str
    scheduled_local: datetime    # naive
    scheduled_tz: str            # 'Asia/Seoul'

def schedule(meeting: Meeting):
    # 검증: tz 이름이 유효한지
    try:
        ZoneInfo(meeting.scheduled_tz)
    except ZoneInfoNotFoundError:
        raise ValueError(f"Unknown timezone: {meeting.scheduled_tz}")
    db.save(meeting)

def get_actual_start_utc(meeting: Meeting) -> datetime:
    tz = ZoneInfo(meeting.scheduled_tz)
    aware_local = meeting.scheduled_local.replace(tzinfo=tz)
    return aware_local.astimezone(timezone.utc)
```

### 6.4 회귀 테스트

```python
import pytest
from datetime import datetime, timezone, timedelta
from zoneinfo import ZoneInfo

def assert_utc_aware(dt: datetime):
    assert dt.tzinfo is not None, "datetime must be timezone-aware"
    assert dt.utcoffset() == timedelta(0), "datetime must be UTC"

def test_now_is_utc_aware():
    now = datetime.now(timezone.utc)
    assert_utc_aware(now)

def test_naive_datetime_rejected_at_boundary():
    naive = datetime(2026, 4, 11, 12, 0)
    with pytest.raises(AssertionError):
        assert_utc_aware(naive)

def test_seoul_aware_rejected():
    seoul = datetime(2026, 4, 11, 12, 0, tzinfo=ZoneInfo("Asia/Seoul"))
    with pytest.raises(AssertionError):
        assert_utc_aware(seoul)
```

---

## 7. 안티패턴 모음

| # | 안티패턴 | 왜 나쁜가 |
|---|---------|----------|
| 1 | DB에 KST/PST 등 약어 저장 | DST/정책 변경 시 박살 |
| 2 | `+09:00` 같은 offset 저장 | DST 못 따라감 |
| 3 | `country` 하나로 timezone+locale 추론 | 미국 사는 한국인 케이스 깨짐 |
| 4 | `datetime.now()` (timezone 인자 X) | OS local timezone 의존, 환경마다 다른 결과 |
| 5 | `datetime.utcnow()` | naive UTC. 다른 aware datetime과 비교 시 TypeError |
| 6 | 도메인 코드 안에서 `astimezone()` | 도메인이 timezone을 알게 됨, 레이어 분리 위반 |
| 7 | 메시지에 변수 직접 박기 (`f"{count}개"`) | 복수형/조사 처리 깨짐 |
| 8 | timezone을 enum으로 정의 | IANA tz는 매년 갱신, 텍스트로 저장해야 함 |
| 9 | 미래 시각을 UTC로 변환 저장 | 정책 변경 시 wall clock 의도 손실 |
| 10 | Frontend가 string 파싱해서 timezone 추측 | 백엔드가 ISO 8601 + tz 명시로 보내야 함 |
| 11 | 로그를 사용자 timezone으로 찍기 | 분산 환경에서 로그 상관관계 깨짐. 무조건 UTC |
| 12 | `babel` 없이 직접 통화 포맷 (`f"${amount:.2f}"`) | 통화별 소수점 자릿수, 자릿수 구분자, 통화 위치 다 다름 |

---

## 8. 실전 함정 사례 (외부 인용)

### 8.1 GitHub의 활동 그래프
GitHub은 한때 사용자 timezone을 추정해서 활동 그래프를 그렸는데, **여행 중인 사용자의 활동이 잘못 표시**되는 문제로 사용자 명시 설정으로 전환.

### 8.2 AWS Lambda의 timezone
Lambda 컨테이너의 default timezone은 UTC. 로컬에서 KST로 테스트하다가 배포 후 시간이 9시간 차이 나는 버그가 흔하다. → **`datetime.now()`를 절대 쓰지 말고 `datetime.now(timezone.utc)` 명시**.

### 8.3 MySQL `TIMESTAMP` vs `DATETIME`
- `TIMESTAMP`: 세션 timezone 기반으로 UTC로 저장 (PostgreSQL의 `TIMESTAMPTZ`와 비슷)
- `DATETIME`: timezone 무시, 입력 그대로 저장 (PostgreSQL의 `TIMESTAMP`)

이름이 PostgreSQL과 반대라 혼란. MySQL 쓸 거면 `TIMESTAMP` (UTC 변환되는 쪽).

### 8.4 JavaScript `Date`의 함정
`new Date("2026-04-11")`는 UTC로 파싱하지만 `new Date("2026-04-11 09:00:00")`는 local timezone으로 파싱. 같은 함수가 입력 형식에 따라 다른 timezone을 적용. → `Temporal` API (stage 3) 또는 `date-fns-tz` 사용 권장.

### 8.5 Google Calendar의 wall clock 저장
Google Calendar는 반복 일정을 wall clock + timezone으로 저장한다. 사용자가 일정을 만든 후 그 도시의 timezone 정책이 바뀌어도, 일정은 원래 의도한 wall clock 시각에 알람이 울린다. iCalendar (RFC 5545)도 같은 패턴.

---

## 9. 결론 (한 줄 요약)

> **3축 분리 (country / timezone / locale), DB는 UTC + IANA, 변환은 edge에서, 미래 시각은 wall + tz, 도메인은 timezone 무지.**

이 원칙을 처음부터 지키면 나중에 다국어/다지역 확장 시 거의 공짜다. 안 지키면 **데이터를 통째로 마이그레이션하면서** 확장해야 한다.

---

## References

- [IANA Time Zone Database](https://www.iana.org/time-zones)
- [Python `zoneinfo` (3.9+)](https://docs.python.org/3/library/zoneinfo.html)
- [Python `datetime` deprecation of `utcnow`](https://docs.python.org/3/library/datetime.html#datetime.datetime.utcnow)
- [BCP 47 Language Tags](https://www.rfc-editor.org/info/bcp47)
- [ISO 3166-1 country codes](https://en.wikipedia.org/wiki/ISO_3166-1)
- [Unicode CLDR](https://cldr.unicode.org/)
- [Babel (Python i18n library)](https://babel.pocoo.org/)
- [Django Time Zones](https://docs.djangoproject.com/en/stable/topics/i18n/timezones/)
- [SQLAlchemy DateTime types](https://docs.sqlalchemy.org/en/20/core/type_basics.html#sqlalchemy.types.DateTime)
- [iCalendar RFC 5545](https://www.rfc-editor.org/rfc/rfc5545)
- [Best practices for timestamps and time zones (Tinybird)](https://www.tinybird.co/blog/database-timestamps-timezones)
- [Just store UTC? Not so fast (CodeOpinion)](https://codeopinion.com/just-store-utc-not-so-fast-handling-time-zones-is-complicated/)
- [Storing timestamps with timezones in PostgreSQL (AboutBits)](https://aboutbits.it/blog/2022-11-08-storing-timestamps-with-timezones-in-postgres)
- [The Falsehoods Programmers Believe About Time](https://infiniteundo.com/post/25326999628/falsehoods-programmers-believe-about-time)
- [Falsehoods about i18n (W3C)](https://www.w3.org/International/articles/falsehoods/)
