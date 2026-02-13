---
name: dan-wiki
description: 개인 Confluence (doheelab.atlassian.net) 페이지 조회/생성/편집
---

# Dan Wiki 스킬

개인 Atlassian Cloud (doheelab.atlassian.net)의 Confluence 페이지를 관리합니다.

## 실행 방법

```
/dan-wiki <자연어 명령>
```

### 예시
```
/dan-wiki 스페이스 목록
/dan-wiki DAN 스페이스의 페이지 목록
/dan-wiki 페이지 12345 조회
/dan-wiki 페이지 생성: 제목 "회의록", 스페이스 DAN
/dan-wiki https://doheelab.atlassian.net/wiki/spaces/DAN/pages/12345 페이지 업데이트
/dan-wiki 페이지 12345에 "새 섹션 내용" 추가
```

## MCP 서버 정보

- **MCP 서버명**: `jira-personal`
- **도구 접두사**: `mcp__jira-personal__*`
- **사이트**: https://doheelab.atlassian.net/
- **이메일**: doheelab@gmail.com

## 동작 절차

사용자의 자연어 명령을 분석하여 아래 중 적절한 작업을 자동으로 수행합니다.

---

### 스페이스 조회

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__get-spaces ()
mcp__jira-personal__find-space-by-name (spaceName)
```

#### curl 대안 (MCP 실패 시)
```bash
# 스페이스 목록
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/space"

# 특정 스페이스
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/space/<SPACE_KEY>"
```

---

### 페이지 조회

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__get-page-by-id (pageId)
mcp__jira-personal__get-page-children (pageId)
mcp__jira-personal__get-pages (spaceKey)
mcp__jira-personal__get-recently-updated-pages ()
```

#### curl 대안
```bash
# 페이지 조회 (본문 포함)
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/content/<PAGE_ID>?expand=version,body.storage"

# 스페이스의 페이지 목록
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/content?spaceKey=<SPACE_KEY>&type=page&limit=25"

# 페이지 검색 (CQL)
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/content/search?cql=space=<SPACE_KEY>+AND+type=page+AND+title~\"<검색어>\""

# 하위 페이지 목록
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/content/<PAGE_ID>/child/page"
```

---

### 페이지 생성

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

### 페이지 업데이트

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__update-page (pageId, title, body, version)
```

#### curl 대안
```bash
# 1. 현재 버전 확인
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/content/<PAGE_ID>?expand=version"

# 2. 페이지 업데이트 (버전 +1 필수)
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

### 페이지 삭제

#### MCP 도구 사용 (우선)
```
mcp__jira-personal__delete-page (pageId)
```

#### curl 대안
```bash
curl -s -X DELETE \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/wiki/rest/api/content/<PAGE_ID>"
```

---

## API 인증 정보

- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회
- **Confluence API**: v1 (Storage Format XHTML 사용)

```bash
# 토큰 조회
cat ~/.claude/settings.json | jq -r '.mcpServers["jira-personal"].env.ATLASSIAN_API_TOKEN'
```

---

## Confluence Storage Format (XHTML)

Confluence는 Storage Format (XHTML)을 사용합니다:

```html
<h2>제목</h2>
<p>문단 내용</p>
<ul>
  <li>항목 1</li>
  <li>항목 2</li>
</ul>
<table>
  <tr><th>헤더1</th><th>헤더2</th></tr>
  <tr><td>값1</td><td>값2</td></tr>
</table>
<ac:structured-macro ac:name="code">
  <ac:parameter ac:name="language">python</ac:parameter>
  <ac:plain-text-body><![CDATA[print("hello")]]></ac:plain-text-body>
</ac:structured-macro>
```

---

## URL에서 페이지 ID 추출

Confluence URL에서 페이지 ID를 추출하는 방법:

```
https://doheelab.atlassian.net/wiki/spaces/DAN/pages/12345/페이지제목
→ pageId: 12345
```

---

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- MCP 도구를 우선 사용하고, 실패 시 curl로 대안 시도
- 작업 완료 후 결과 URL을 반드시 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- 페이지 업데이트 시 **버전 번호를 반드시 1 증가**
- 페이지 내용을 수정할 때는 기존 내용을 먼저 조회한 후 수정
