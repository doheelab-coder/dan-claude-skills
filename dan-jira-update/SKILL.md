---
name: dan-jira-update
description: Jira TASK 프로젝트 이슈 상태를 날짜 기준으로 자동 전환
---

# Dan Jira Update 스킬

TASK 프로젝트의 이슈 상태를 제목의 날짜 `(M/DD)` 기준으로 자동 전환합니다.

- **In Progress** 중 오늘 이전 날짜 → **Done**
- **To Do** 중 오늘 날짜 → **In Progress**

## 실행 방법

```
/dan-jira-update
```

인자 없이 실행하면 오늘 날짜 기준으로 자동 처리합니다.

## MCP 서버 정보

- **MCP 서버명**: `jira-personal`
- **도구 접두사**: `mcp__jira-personal__*`
- **사이트**: https://doheelab.atlassian.net/
- **이메일**: doheelab@gmail.com

## 동작 절차

### 1단계: 오늘 날짜 확인

오늘 날짜를 `M/DD` 형식으로 준비합니다 (예: `2/14`, `12/3`).

- 앞에 0을 붙이지 않음: `2/14` (O), `02/14` (X)
- 현재 연도를 기준으로 날짜 비교

### 2단계: 활성 스프린트 이슈 전체 조회

활성 스프린트의 모든 이슈를 한 번에 조회합니다.

**JQL**: `project=TASK AND sprint in openSprints() AND assignee=currentUser() AND issuetype=Task ORDER BY status ASC, updated DESC`

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__search-issues (jql)
```

#### curl 대안 (MCP 실패 시)
```bash
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/search/jql?jql=project=TASK+AND+sprint+in+openSprints()+AND+assignee=currentUser()+AND+issuetype=Task+ORDER+BY+status+ASC,updated+DESC&maxResults=50&fields=key,summary,status"
```

### 3단계: 상태별 전환 처리

조회된 이슈를 상태 카테고리(`statusCategory.key`)로 분류하여 처리합니다.

**상태명 매핑** (한글/영어 모두 대응):
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

#### 해야 할 일 이슈 → 진행중 처리

`statusCategory.key == "new"` 인 이슈 중:

1. **제목에서 날짜 추출**: 정규식 `\((\d{1,2}/\d{1,2})\)` 매칭
2. **날짜가 없으면 건너뜀**
3. **오늘 날짜와 일치하는 경우 진행중으로 전환**:
   - transitions 조회 → "진행중" 또는 "In Progress" transition ID 확인
   - transition 실행

### 4단계: 결과 보고

전환 결과를 요약하여 출력합니다:

```
## 완료 처리 (In Progress → Done)
- TASK-101: (2/13) 서윤정님
- TASK-102: (2/12) 운동

## 진행 시작 (To Do → In Progress)
- TASK-105: (2/14) 김하나님
- TASK-106: (2/14) 독서

## 건너뜀 (날짜 없음)
- TASK-110: API 리팩토링
```

## 날짜 비교 규칙

- 제목의 `(M/DD)` 패턴을 현재 연도 기준 날짜로 해석
- **연도 전환**: 현재 1~2월이고 제목 날짜가 11~12월이면 전년도로 간주
- **오늘 이전** = 제목 날짜 < 오늘 (오늘 포함하지 않음)
- **오늘** = 제목 날짜 == 오늘
- 날짜 패턴이 없는 이슈는 건너뜀

## API 인증 정보

- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회

```bash
cat ~/.claude/settings.json | jq -r '.mcpServers["jira-personal"].env.ATLASSIAN_API_TOKEN'
```

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- MCP 도구를 우선 사용하고, 실패 시 curl로 대안 시도
- 전환 전 반드시 transitions 목록을 조회하여 올바른 ID 사용
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- 전환 대상이 없으면 "전환할 이슈가 없습니다"로 보고
