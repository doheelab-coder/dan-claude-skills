---
name: dan-calendar
description: 개인 Google Calendar 일정 조회/생성/수정/삭제
---

# Dan Calendar 스킬

개인 Google Calendar (doheelab@gmail.com)의 일정을 관리합니다.

## 실행 방법

```
/dan-calendar <자연어 명령>
```

### 예시
```
/dan-calendar 오늘 일정
/dan-calendar 이번 주 일정
/dan-calendar 2/15 오후 3시 PT 일정 추가
/dan-calendar 내일 7시 약속 일정 삭제
```

## 인증 정보

- **인증 방식**: Google OAuth2
- **인증 정보 위치**: `/Users/dan.jung/Documents/code/personal/token.txt`
- **Calendar ID**: `primary` (doheelab@gmail.com)

```bash
# 인증 정보는 token.txt에서 읽기
TOKEN_FILE="/Users/dan.jung/Documents/code/personal/token.txt"
GCAL_CLIENT_ID=$(grep "Client ID:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
GCAL_CLIENT_SECRET=$(grep "Client Secret:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
GCAL_REFRESH_TOKEN=$(grep "Refresh Token:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
```

## 동작 절차

**모든 API 호출 전에 반드시 Access Token을 갱신해야 합니다.**

### Step 1: Access Token 갱신 (필수)

```bash
ACCESS_TOKEN=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=${GCAL_CLIENT_ID}&client_secret=${GCAL_CLIENT_SECRET}&refresh_token=${GCAL_REFRESH_TOKEN}&grant_type=refresh_token" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

### Step 2: API 호출

---

### 일정 조회

```bash
# 특정 기간 일정 조회 (timeMin, timeMax는 RFC3339 형식)
curl -s "https://www.googleapis.com/calendar/v3/calendars/primary/events?timeMin=<START>&timeMax=<END>&singleEvents=true&orderBy=startTime&maxResults=20" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" | python3 -m json.tool

# 오늘 일정 (예시)
# timeMin=2026-02-13T00:00:00%2B09:00&timeMax=2026-02-14T00:00:00%2B09:00

# 이번 주 일정 (월~일)
# timeMin=2026-02-09T00:00:00%2B09:00&timeMax=2026-02-16T00:00:00%2B09:00
```

**참고**: URL에서 `+09:00`은 `%2B09:00`으로 인코딩해야 합니다.

---

### 일정 생성

**생성 전 중복 체크 (필수):**

일정을 생성하기 전에 같은 날짜의 기존 일정을 조회하여 중복 여부를 확인합니다.

1. 생성할 일정의 날짜 범위로 기존 일정 조회
2. 동일한 `summary`(제목) 또는 `description`(TASK-XXX)이 이미 존재하면:
   - 새로 생성하지 않고, 기존 일정을 **수정(PATCH)**하여 업데이트
3. 중복이 없을 때만 새 일정 생성

```bash
# 중복 체크: 해당 날짜의 기존 일정 조회
curl -s "https://www.googleapis.com/calendar/v3/calendars/primary/events?timeMin=<DATE>T00:00:00%2B09:00&timeMax=<DATE+1>T00:00:00%2B09:00&singleEvents=true&orderBy=startTime" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
# → items에서 summary 또는 description이 일치하는 이벤트가 있으면 PATCH, 없으면 POST
```

```bash
curl -s -X POST "https://www.googleapis.com/calendar/v3/calendars/primary/events" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "summary": "<일정 제목>",
    "location": "<장소>",
    "description": "<설명>",
    "start": {
      "dateTime": "2026-02-15T15:00:00+09:00",
      "timeZone": "Asia/Seoul"
    },
    "end": {
      "dateTime": "2026-02-15T16:00:00+09:00",
      "timeZone": "Asia/Seoul"
    },
    "reminders": {
      "useDefault": false,
      "overrides": [
        {"method": "popup", "minutes": 30}
      ]
    }
  }'

# 종일 일정 (시간 없이 date만 사용)
curl -s -X POST "https://www.googleapis.com/calendar/v3/calendars/primary/events" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "summary": "<일정 제목>",
    "start": {"date": "2026-02-15"},
    "end": {"date": "2026-02-16"}
  }'
```

---

### 일정 수정

```bash
curl -s -X PATCH "https://www.googleapis.com/calendar/v3/calendars/primary/events/<EVENT_ID>" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "summary": "<수정된 제목>",
    "start": {
      "dateTime": "2026-02-15T16:00:00+09:00",
      "timeZone": "Asia/Seoul"
    },
    "end": {
      "dateTime": "2026-02-15T17:00:00+09:00",
      "timeZone": "Asia/Seoul"
    }
  }'
```

---

### 일정 삭제

```bash
curl -s -X DELETE "https://www.googleapis.com/calendar/v3/calendars/primary/events/<EVENT_ID>" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

---

## 일정 작성 규칙

- **일정명(summary)**: Jira 이슈 제목에서 날짜 부분(`(M/DD)` 등)을 제거하고 등록
  - 예: `(2/15) 이제연님` → summary: `이제연님`
- **시간대**: 항상 `Asia/Seoul` (+09:00) 사용
- **end 시간**: 지정하지 않은 경우 start로부터 1시간 후로 설정
- **연애 에픽 일정 (colorId 4)**: 항상 start로부터 **2시간** 짜리 일정으로 생성 (summary에 `님` 포함 또는 연애 에픽)
- **알림**: 기본 30분 전 팝업 알림
- **종일 일정**: `dateTime` 대신 `date` 필드 사용

---

## Jira 연동 규칙

### Jira → Calendar (dan-jira에서 이슈 변경 시)

- `/dan-jira`로 이슈를 생성/수정/삭제할 때, 해당 이슈에 날짜 정보가 있으면 Google Calendar도 함께 동기화
  - 이슈 생성 → 캘린더 일정 생성
  - 이슈 수정 (날짜/시간/장소 변경) → 캘린더 일정 수정
  - 이슈 삭제 → 캘린더 일정 삭제

### Calendar → Jira (dan-calendar에서 일정 추가 시)

- `/dan-calendar`로 일정을 **추가**할 때, Jira TASK 이슈도 함께 생성하고 활성 스프린트에 추가
- 절차:
  1. **Jira 이슈 생성**: project `TASK`, issuetype `Task` (id: 10003)
     - 제목: `(M/DD) {일정명}` (예: `(2/18) 복이안님`)
     - description: 시간, 장소 등 상세 정보 (ADF 형식)
     - 사람 이름(`님` 포함)인 경우: Confluence 위키에서 `{이름} 프로필` 페이지 검색 → 존재하면 description에 링크 추가
  2. **활성 스프린트에 이동**: Jira Agile API로 현재 활성 스프린트에 이슈 추가
     ```bash
     # 활성 스프린트 조회 (board ID: 1)
     curl -s "https://doheelab.atlassian.net/rest/agile/1.0/board/1/sprint?state=active" \
       -u "doheelab@gmail.com:<API_TOKEN>"
     # 스프린트에 이슈 추가
     curl -s -X POST "https://doheelab.atlassian.net/rest/agile/1.0/sprint/<SPRINT_ID>/issue" \
       -u "doheelab@gmail.com:<API_TOKEN>" \
       -H "Content-Type: application/json" \
       -d '{"issues": ["TASK-XXX"]}'
     ```
  3. **Google Calendar 일정 생성**: description에 `TASK-XXX` 이슈 키 포함
- 일정 **조회/수정/삭제** 시에는 Jira 이슈를 생성하지 않음

### 공통 규칙

- **매칭 기준**: 캘린더 이벤트의 description에 `TASK-XXX` 형식의 이슈 키를 포함하여 매칭
- **에픽별 색상**: Jira 이슈의 에픽에 따라 캘린더 이벤트의 `colorId`를 설정
- **API 인증**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 토큰 조회

### 에픽-색상 매핑 (Google Calendar colorId)

| colorId | 색상 | 에픽 |
|---------|------|------|
| 1 | Lavender (연보라) | [목적] 독서 |
| 2 | Sage (초록) | [신체] 운동 |
| 3 | Grape (보라) | [관계] 마술 |
| 4 | Flamingo (핑크) | [관계] 연애 |
| 5 | Banana (노랑) | [관계] 친구 |
| 6 | Tangerine (주황) | [관계] 가족 |
| 7 | Peacock (파랑) | [목적] 업무 |
| 8 | Graphite (회색) | (미사용) |
| 9 | Blueberry (남색) | [목적] 스터디 |
| 10 | Basil (짙은초록) | [목적] 방향성 |
| 11 | Tomato (빨강) | [경제] 재테크, [경제] 투자 |

---

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- 모든 API 호출 전 반드시 Access Token 갱신
- 작업 완료 후 결과를 사용자에게 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
