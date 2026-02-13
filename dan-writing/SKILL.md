---
name: dan-writing
description: 개인 Jira/Confluence (doheelab.atlassian.net) 이슈 및 페이지 편집
---

# Dan Writing 스킬

개인 Atlassian Cloud (doheelab.atlassian.net)의 Jira 이슈와 Confluence 페이지를 편집합니다.

## 실행 방법

```
/dan-writing <자연어 명령>
```

### 예시
```
/dan-writing DAN-1 이슈에 코멘트 추가: 오늘 작업 완료
/dan-writing DAN-1 이슈 Description 업데이트
/dan-writing 새 이슈 생성: 프로젝트 DAN, 제목 "API 연동 작업"
/dan-writing Confluence 페이지 생성: 제목 "회의록", 스페이스 DAN
/dan-writing https://doheelab.atlassian.net/wiki/spaces/DAN/pages/12345 페이지 업데이트
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
```

#### curl 대안 (MCP 실패 시)
```bash
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>"
```

---

### Jira 이슈 생성

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__create-issue (projectKey, issueType, summary, description)
```

#### curl 대안
```bash
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

### Confluence 페이지 조회

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__get-page-by-id (pageId)
mcp__jira-personal__get-spaces ()
```

#### curl 대안
```bash
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/content/<PAGE_ID>?expand=version,body.storage"
```

---

### Confluence 페이지 생성

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__create-page (spaceKey, title, body, parentPageId)
```

#### curl 대안
```bash
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/wiki/rest/api/content" \
  -d '{
    "type": "page",
    "title": "<제목>",
    "space": {"key": "<SPACE_KEY>"},
    "ancestors": [{"id": "<부모페이지ID>"}],
    "body": {
      "storage": {
        "value": "<HTML 내용>",
        "representation": "storage"
      }
    }
  }'
```

---

### Confluence 페이지 업데이트

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__update-page (pageId, title, body, version)
```

#### curl 대안
```bash
curl -s -X PUT \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/wiki/rest/api/content/<PAGE_ID>" \
  -d '{
    "type": "page",
    "title": "<제목>",
    "version": {"number": <현재버전+1>},
    "body": {
      "storage": {
        "value": "<HTML 내용>",
        "representation": "storage"
      }
    }
  }'
```

---

## API 인증 정보

- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회
- **Jira API**: v3 (Cloud 전용, ADF 형식 사용)
- **Confluence API**: v1 (Storage Format XHTML 사용)

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

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- MCP 도구를 우선 사용하고, 실패 시 curl로 대안 시도
- 작업 완료 후 결과 URL을 반드시 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- Confluence 페이지 업데이트 시 **버전 번호를 반드시 1 증가**
