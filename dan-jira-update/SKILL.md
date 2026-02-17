---
name: dan-jira-update
description: Jira 이슈 상태 자동 전환 + Google Calendar 일정 검증/수정
---

# Dan Jira Update 스킬

TASK 프로젝트 이슈와 Google Calendar 일정을 자동으로 검증/수정합니다.

**Jira 이슈 전환:**
- **In Progress** 중 오늘 이전 날짜 → **Done**
- **To Do** 중 오늘 이전 날짜 → **Done**
- **To Do** 중 오늘 날짜 → **In Progress**
- **모든 상태**의 이슈를 날짜순으로 정렬 (같은 날짜 내 [관계] 에픽 우선)

**Google Calendar 검증:**
- 색상(colorId) 검증 및 수정
- 시간 검증 및 수정
- summary 정리

## 실행 방법

```
/dan-jira-update
```

인자 없이 실행하면 오늘 날짜 기준으로 자동 처리합니다.

---

## Part 1: Jira 이슈 상태 전환

### 1단계: 오늘 날짜 확인

오늘 날짜를 `M/DD` 형식으로 준비합니다 (예: `2/14`, `12/3`).

- 앞에 0을 붙이지 않음: `2/14` (O), `02/14` (X)
- 현재 연도를 기준으로 날짜 비교

### 2단계: 활성 스프린트 이슈 전체 조회

활성 스프린트의 모든 이슈를 한 번에 조회합니다.

**JQL**: `project=TASK AND sprint in openSprints() AND assignee=currentUser() AND issuetype=Task ORDER BY status ASC, updated DESC`

```bash
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/search/jql?jql=project=TASK+AND+sprint+in+openSprints()+AND+assignee=currentUser()+AND+issuetype=Task+ORDER+BY+status+ASC,updated+DESC&maxResults=50&fields=key,summary,status,parent"
```

### 3단계: 상태별 전환 처리

조회된 이슈를 상태 카테고리(`statusCategory.key`)로 분류하여 처리합니다.

**상태명 매핑**:
- `statusCategory.key = "new"` → 해야 할 일 (To Do)
- `statusCategory.key = "indeterminate"` → 진행중 (In Progress)
- `statusCategory.key = "done"` → 완료 (Done)

#### 진행중 이슈 → 완료 처리

`statusCategory.key == "indeterminate"` 인 이슈 중:

1. **제목에서 날짜 추출**: 정규식 `\((\d{1,2}/\d{1,2})\)` 매칭
2. **날짜가 없으면 건너뜀**
3. **오늘 이전 날짜인 경우 완료로 전환**:
   - transitions 조회 → "완료" 또는 "Done" transition ID 확인
   - transition 실행

#### 해야 할 일 이슈 → 완료 또는 진행중 처리

`statusCategory.key == "new"` 인 이슈 중:

1. **제목에서 날짜 추출**: 정규식 `\((\d{1,2}/\d{1,2})\)` 매칭
2. **날짜가 없으면 건너뜀**
3. **오늘 이전 날짜인 경우 완료로 전환**:
   - transitions 조회 → "완료" 또는 "Done" transition ID 확인
   - transition 실행
4. **오늘 날짜와 일치하는 경우 진행중으로 전환**:
   - transitions 조회 → "진행중" 또는 "In Progress" transition ID 확인
   - transition 실행

### 4단계: 전체 이슈 날짜순 정렬

모든 상태(To Do, In Progress, Done)의 이슈를 제목의 `(M/DD)` 날짜 기준으로 오름차순 정렬합니다.

**정렬 규칙**:
1. **날짜 오름차순**: 이른 날짜가 위
2. **같은 날짜 내 [관계] 에픽 우선**: [관계] 에픽(TASK-4, TASK-5, TASK-6, TASK-16) 또는 `님` 포함 이슈가 먼저
3. 날짜 패턴이 없는 이슈는 맨 아래

```bash
# Agile rank API로 이슈 순서 변경
curl -s -X PUT -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/agile/1.0/issue/rank" \
  -d '{"issues":["TASK-XXX"],"rankAfterIssue":"TASK-YYY"}'
```

### 5단계: 사람 이슈에 Confluence 프로필 링크 추가

제목에 사람 이름이 포함된 이슈 (정규식 `\)\s*(.+?)님`)에 대해:

1. **Confluence 위키에서 프로필 페이지 검색**: `{이름} 프로필` 제목으로 검색
   - 스페이스 키: `~6167d0ad07ac3c00689b46b0`
   - 부모 페이지: 관계 (ID: 40370179)
   - **중요**: CQL 검색 결과를 `title == "{이름} 프로필"`로 정확히 필터링할 것 (CQL `title=`은 부분 매칭이므로 코드에서 exact match 필수)
2. **프로필이 존재하면**: 해당 이슈의 description에 위키 링크 추가
   - 기존 description 유지 + `프로필: {이름} 프로필` 링크 행 추가
   - 이미 링크가 있으면 건너뜀
3. **프로필이 없으면**: 건너뜀

#### Confluence 인증
- `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN` + email (`doheelab@gmail.com`)
- Basic Auth (base64), python3 urllib 사용
- **반드시 메인 컨텍스트에서 직접 실행** (서브에이전트 위임 금지)

---

## Part 2: Google Calendar 일정 검증/수정

### 6단계: Google Calendar Access Token 갱신

```bash
TOKEN_FILE="/Users/dan.jung/Documents/code/personal/token.txt"
GCAL_CLIENT_ID=$(grep "Client ID:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
GCAL_CLIENT_SECRET=$(grep "Client Secret:" "$TOKEN_FILE" | head -1 | awk '{print $3}')
GCAL_REFRESH_TOKEN=$(grep "Refresh Token:" "$TOKEN_FILE" | head -1 | awk '{print $3}')

ACCESS_TOKEN=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=${GCAL_CLIENT_ID}&client_secret=${GCAL_CLIENT_SECRET}&refresh_token=${GCAL_REFRESH_TOKEN}&grant_type=refresh_token" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

### 7단계: 오늘부터 2주간 일정 조회

```bash
curl -s "https://www.googleapis.com/calendar/v3/calendars/primary/events?timeMin=<START>&timeMax=<END>&singleEvents=true&orderBy=startTime&maxResults=50" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```

### 8단계: 검증 규칙 적용

#### 8-1. 색상(colorId) 검증

일정의 summary를 기반으로 올바른 에픽 색상이 지정되어 있는지 확인합니다.

**자동 추론 규칙**:
- summary에 `님` → colorId `4` (Flamingo/연애)
- summary에 `PT`, `피티`, `러닝`, `레이스`, `운동` → colorId `2` (Sage/운동)
- summary에 `제사`, `명절`, `할머니`, `부모님` → colorId `6` (Tangerine/가족)
- description에 Jira 이슈 키가 있으면 → 해당 이슈의 에픽으로 색상 결정

#### 8-2. 시간 검증

- summary에 시간 힌트가 있으면 (예: `2시 복이안님`) 실제 시작 시간과 비교
  - `N시` → N:00 (1~6 → 오후 13:00~18:00, 7~12 → 오전/오후 판단)
- 시작 시간이 08:00이고 summary에 시간 힌트가 있으면 → 수동 입력 오류로 간주, 수정
- 연애 에픽(님 패턴) duration이 2시간이 아니면 → 2시간으로 수정
- 운동 에픽 duration이 1시간이 아니면 → 1시간으로 수정

#### 8-3. summary 정리

- summary에 시간 정보가 포함되어 있으면 제거: `2시 복이안님` → `복이안님`
- summary에 날짜 패턴 `(M/DD)` 가 있으면 제거

### 9단계: 수정 실행

검증에서 문제가 발견된 일정을 PATCH API로 수정합니다.

```bash
curl -s -X PATCH "https://www.googleapis.com/calendar/v3/calendars/primary/events/<EVENT_ID>" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{ ... 수정 내용 ... }'
```

---

## 10단계: 결과 보고

전체 결과를 통합하여 보고합니다:

```
## Jira 이슈 전환

### 완료 처리 (In Progress → Done)
- TASK-101: (2/13) 서윤정님

### 완료 처리 (To Do → Done)
- TASK-155: (2/15) 달리기

### 진행 시작 (To Do → In Progress)
- TASK-105: (2/14) 김하나님

### 이슈 정렬 (날짜순, [관계] 우선)
#### To Do
- TASK-154 (2/17) [관계] → TASK-135 (2/18) [관계] → TASK-142 (2/22)
#### In Progress
- TASK-158 (2/16) [관계] → TASK-157 (2/16) [관계] → TASK-144 (2/16)
#### Done
- TASK-137 (2/13) [관계] → ... → TASK-155 (2/15)

### 프로필 링크 추가
- TASK-105: 김하나님 → 김하나 프로필 (Confluence) 링크 추가

### 건너뜀 (날짜 없음)
- TASK-110: API 리팩토링

## Google Calendar 검증

### 수정된 일정
- 복이안님 (2/17): 시간 08:00→14:00, 색상 →Flamingo(연애), summary 정리

### 정상 일정 (수정 불필요)
- 이제연님 (2/15): colorId=4 ✓, 시간 12:00 ✓

### 색상 추론 불가 (수동 확인 필요)
- 일정명 (날짜): 에픽을 판단할 수 없음
```

## API 인증 정보

### Jira (Atlassian Cloud)
- **인증 방식**: Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회
- **API 호출**: 항상 curl(python3 urllib) 사용 (MCP 도구 사용하지 않음)

### Google Calendar
- **인증 방식**: OAuth2
- **인증 정보**: `/Users/dan.jung/Documents/code/personal/token.txt`
- **Calendar ID**: `primary` (doheelab@gmail.com)

## 날짜 비교 규칙

- 제목의 `(M/DD)` 패턴을 현재 연도 기준 날짜로 해석
- **연도 전환**: 현재 1~2월이고 제목 날짜가 11~12월이면 전년도로 간주
- **오늘 이전** = 제목 날짜 < 오늘 (오늘 포함하지 않음)
- **오늘** = 제목 날짜 == 오늘
- 날짜 패턴이 없는 이슈는 건너뜀

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- **항상 curl(python3 urllib)로 API 호출** (MCP 도구 사용하지 않음)
- 전환 전 반드시 transitions 목록을 조회하여 올바른 ID 사용
- 색상을 자동 추론할 수 없는 경우 수동 확인 필요로 보고 (임의 수정 금지)
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- 전환 대상이 없으면 "전환할 이슈가 없습니다"로 보고
