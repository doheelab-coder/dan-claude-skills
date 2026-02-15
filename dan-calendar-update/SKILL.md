---
name: dan-calendar-update
description: Google Calendar 일정을 dan-calendar 규칙에 맞게 자동 검증/수정
---

# Dan Calendar Update 스킬

Google Calendar의 일정을 `dan-calendar` 스킬의 작성 규칙에 맞게 검증하고, 잘못된 항목을 자동 수정합니다.

## 실행 방법

```
/dan-calendar-update
/dan-calendar-update 이번 주
/dan-calendar-update 2/15 ~ 2/28
```

인자 없이 실행하면 오늘부터 2주간의 일정을 검증합니다.

## 인증 정보

`dan-calendar` 스킬과 동일:

- **인증 방식**: Google OAuth2
- **인증 정보 위치**: `/Users/dan.jung/Documents/code/personal/token.txt`
- **Calendar ID**: `primary` (doheelab@gmail.com)

```bash
TOKEN_FILE="/Users/dan.jung/Documents/code/personal/token.txt"
GCAL_CLIENT_ID=$(grep "Client ID:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
GCAL_CLIENT_SECRET=$(grep "Client Secret:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
GCAL_REFRESH_TOKEN=$(grep "Refresh Token:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
```

## 동작 절차

### Step 1: Access Token 갱신 (필수)

```bash
ACCESS_TOKEN=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=${GCAL_CLIENT_ID}&client_secret=${GCAL_CLIENT_SECRET}&refresh_token=${GCAL_REFRESH_TOKEN}&grant_type=refresh_token" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

### Step 2: 일정 조회

대상 기간의 모든 일정을 조회합니다.

```bash
curl -s "https://www.googleapis.com/calendar/v3/calendars/primary/events?timeMin=<START>&timeMax=<END>&singleEvents=true&orderBy=startTime&maxResults=50" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

### Step 3: 검증 규칙 적용

각 일정에 대해 다음 규칙을 검증합니다:

#### 3-1. 색상(colorId) 검증

일정의 summary(제목)을 기반으로 올바른 에픽 색상이 지정되어 있는지 확인합니다.

**에픽-색상 매핑 (Google Calendar colorId)**:

| colorId | 색상 | 키워드/에픽 |
|---------|------|------------|
| 1 | Lavender (연보라) | [목적] 독서 - 책 제목 관련 |
| 2 | Sage (초록) | [신체] 운동 - PT, 러닝, 레이스 등 |
| 3 | Grape (보라) | [관계] 마술 |
| 4 | Flamingo (핑크) | [관계] 연애 - `~님` 패턴 (사람 이름) |
| 5 | Banana (노랑) | [관계] 친구 |
| 6 | Tangerine (주황) | [관계] 가족 - 제사, 명절, 부모님 등 |
| 7 | Peacock (파랑) | [목적] 업무 |
| 8 | Graphite (회색) | (미사용) |
| 9 | Blueberry (남색) | [목적] 스터디 |
| 10 | Basil (짙은초록) | [목적] 방향성 |
| 11 | Tomato (빨강) | [경제] 재테크, [경제] 투자 |

**자동 추론 규칙**:
- summary에 `님`이 포함되면 → colorId `4` (연애)
- summary에 `PT`, `피티`, `러닝`, `레이스`, `운동` 포함 → colorId `2` (운동)
- summary에 `제사`, `명절`, `할머니`, `부모님` 포함 → colorId `6` (가족)
- description에 Jira 이슈 키가 있으면 → 해당 이슈의 에픽으로 색상 결정

#### 3-2. 시간 검증

- summary에 시간 힌트가 있으면 (예: `2시 복이안님`, `1시 피티`) 실제 시작 시간과 비교
  - `N시` → N:00 (1~6 → 오후 13:00~18:00, 7~12 → 오전/오후 판단)
- 시작 시간이 08:00이고 summary에 시간 힌트가 있으면 → 수동 입력 오류로 간주, 수정
- 연애 에픽(님 패턴) 일정의 duration이 2시간이 아니면 → 2시간으로 수정
- 운동 에픽 일정의 duration이 1시간이 아니면 → 1시간으로 수정

#### 3-3. summary 정리

- summary에 시간 정보가 포함되어 있으면 제거: `2시 복이안님` → `복이안님`
- summary에 날짜 패턴 `(M/DD)` 가 있으면 제거

### Step 4: 수정 실행

검증에서 문제가 발견된 일정을 PATCH API로 수정합니다.

```bash
curl -s -X PATCH "https://www.googleapis.com/calendar/v3/calendars/primary/events/<EVENT_ID>" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{ ... 수정 내용 ... }'
```

### Step 5: 결과 보고

```
## 수정된 일정
- 복이안님 (2/17): 시간 08:00→14:00, 색상 →Flamingo(연애), summary 정리
- PT (2/18): 시간 08:00→13:00, 색상 →Sage(운동)

## 정상 일정 (수정 불필요)
- 이제연님 (2/15): colorId=4 ✓, 시간 12:00 ✓
- 신주현님 (2/18): colorId=4 ✓, 시간 19:30 ✓

## 색상 추론 불가 (수동 확인 필요)
- 일정명 (날짜): 에픽을 판단할 수 없음
```

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- 모든 API 호출 전 반드시 Access Token 갱신
- 수정 전 기존 일정 정보를 출력하여 변경 내역을 명확히 보고
- 색상을 자동 추론할 수 없는 경우 수동 확인 필요로 보고 (임의 수정 금지)
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
