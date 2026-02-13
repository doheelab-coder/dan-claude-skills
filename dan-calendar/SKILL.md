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

- **시간대**: 항상 `Asia/Seoul` (+09:00) 사용
- **end 시간**: 지정하지 않은 경우 start로부터 1시간 후로 설정
- **연애 에픽 일정**: Jira 연애 에픽 하위 이슈를 동기화할 때는 start로부터 **3시간** 짜리 일정으로 생성
- **알림**: 기본 30분 전 팝업 알림
- **종일 일정**: `dateTime` 대신 `date` 필드 사용

---

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- 모든 API 호출 전 반드시 Access Token 갱신
- 작업 완료 후 결과를 사용자에게 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
