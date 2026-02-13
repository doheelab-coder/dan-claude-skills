---
name: dan-jira
description: 개인 Jira (doheelab.atlassian.net) 이슈 조회/생성/편집/코멘트
---

# Dan Jira 스킬

개인 Atlassian Cloud (doheelab.atlassian.net)의 Jira 이슈를 관리합니다.

## 실행 방법

```
/dan-jira <자연어 명령>
```

### 예시
```
/dan-jira DAN-1 이슈 조회
/dan-jira DAN-1 이슈에 코멘트 추가: 오늘 작업 완료
/dan-jira DAN-1 이슈 Description 업데이트
/dan-jira 새 이슈 생성: 프로젝트 DAN, 제목 "API 연동 작업"
/dan-jira 내 이슈 목록
/dan-jira DAN 프로젝트 스프린트 정보
```

## MCP 서버 정보

- **MCP 서버명**: `jira-personal`
- **도구 접두사**: `mcp__jira-personal__*`
- **사이트**: https://doheelab.atlassian.net/
- **이메일**: doheelab@gmail.com

## 동작 절차

사용자의 자연어 명령을 분석하여 아래 중 적절한 작업을 자동으로 수행합니다.

---

### Jira 이슈 조회

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__get-issue (issueKey)
mcp__jira-personal__get-comments (issueKey)
mcp__jira-personal__search-issues (jql)
```

#### curl 대안 (MCP 실패 시)
```bash
# 단일 이슈 조회
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>"

# 이슈 코멘트 조회
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/comment"

# JQL 검색 (/rest/api/3/search는 deprecated → /rest/api/3/search/jql 사용)
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/search/jql?jql=<JQL>&maxResults=20"

# 내 이슈 목록
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/search/jql?jql=assignee=currentUser()+ORDER+BY+updated+DESC&maxResults=20"
```

---

### Jira 보드/스프린트 조회

#### curl (Agile REST API)
```bash
# 보드 목록
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/agile/1.0/board"

# 특정 보드의 스프린트 목록
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/agile/1.0/board/<BOARD_ID>/sprint"

# 활성 스프린트의 이슈
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/agile/1.0/sprint/<SPRINT_ID>/issue?fields=key,summary,status,priority,assignee&maxResults=50"

# 스프린트 닫기
curl -s -X POST -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/agile/1.0/sprint/<SPRINT_ID>" \
  -d '{"state":"closed"}'

# 스프린트 생성
curl -s -X POST -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/agile/1.0/sprint" \
  -d '{
    "name": "<스프린트명>",
    "startDate": "2026-02-01T00:00:00.000+09:00",
    "endDate": "2026-02-28T23:59:59.000+09:00",
    "originBoardId": 1
  }'

# 스프린트 시작 (future → active)
curl -s -X POST -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/agile/1.0/sprint/<SPRINT_ID>" \
  -d '{"state":"active"}'

# 스프린트에 이슈 추가
curl -s -X POST -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/agile/1.0/sprint/<SPRINT_ID>/issue" \
  -d '{"issues":["TASK-XXX","TASK-YYY"]}'

# 이슈 순서 정렬 (rank)
curl -s -X PUT -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/agile/1.0/issue/rank" \
  -d '{"issues":["TASK-XXX"],"rankAfterIssue":"TASK-YYY"}'
```

---

### Jira 이슈 생성

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__create-issue (projectKey, issueType, summary, description)
```

#### curl 대안
```bash
# 기본 이슈 생성
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": {"key": "<PROJECT_KEY>"},
      "issuetype": {"name": "Task"},
      "summary": "<제목>",
      "description": {
        "type": "doc",
        "version": 1,
        "content": [{"type": "paragraph", "content": [{"type": "text", "text": "<내용>"}]}]
      }
    }
  }'

# 에픽 하위 태스크 생성 (parent 필드로 에픽 연결)
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": {"key": "<PROJECT_KEY>"},
      "issuetype": {"name": "Task"},
      "parent": {"key": "<EPIC_KEY>"},
      "summary": "<제목>",
      "description": {
        "type": "doc",
        "version": 1,
        "content": [{"type": "paragraph", "content": [{"type": "text", "text": "<내용>"}]}]
      }
    }
  }'
```

---

### Jira 이슈 편집 (Description 업데이트)

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__edit-issue (issueKey, fields)
```

#### curl 대안
```bash
curl -s -X PUT \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>" \
  -d '{
    "fields": {
      "description": {
        "type": "doc",
        "version": 1,
        "content": [{"type": "paragraph", "content": [{"type": "text", "text": "<내용>"}]}]
      }
    }
  }'
```

---

### Jira 코멘트 추가

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__create-comment (issueKey, body)
```

#### curl 대안
```bash
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/comment" \
  -d '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [{"type": "paragraph", "content": [{"type": "text", "text": "<내용>"}]}]
    }
  }'
```

---

### Jira 이슈 상태 변경

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__get-issue-transitions (issueKey)
mcp__jira-personal__transition-issue (issueKey, transitionId)
```

#### curl 대안
```bash
# 가능한 전환 목록 조회
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/transitions"

# 상태 전환
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/transitions" \
  -d '{"transition": {"id": "<TRANSITION_ID>"}}'
```

---

## API 인증 정보

- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회
- **Jira API**: v3 (Cloud 전용, ADF 형식 사용)

```bash
# 토큰 조회
cat ~/.claude/settings.json | jq -r '.mcpServers["jira-personal"].env.ATLASSIAN_API_TOKEN'
```

---

## Atlassian Document Format (ADF) - Jira Cloud

Jira Cloud API v3는 ADF 형식을 사용합니다:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "paragraph",
      "content": [{"type": "text", "text": "내용"}]
    },
    {
      "type": "heading",
      "attrs": {"level": 2},
      "content": [{"type": "text", "text": "제목"}]
    },
    {
      "type": "bulletList",
      "content": [
        {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "항목"}]}]}
      ]
    }
  ]
}
```

---

## 이슈 작성 규칙

- **제목**: 일자와 핵심 정보만 포함 (예: `(2/13) 서윤정님`)
- **본문(Description)**: 시간, 장소 등 상세 정보 기재 (예: `18:00 이수 방콕상회`)
- 시간 정보는 제목에 넣지 않고 본문에 넣기

---

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- MCP 도구를 우선 사용하고, 실패 시 curl로 대안 시도
- 작업 완료 후 결과 URL을 반드시 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
