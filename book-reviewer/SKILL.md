---
name: book-reviewer
description: 책 제목으로 독서리뷰를 자동 생성하고 Jira/Confluence에 등록
---

# Book Reviewer 스킬

책 제목만 입력하면 WebSearch로 책 정보를 수집하고, 구조화된 독서리뷰를 생성한 뒤, Jira 태스크 + Confluence 위키 업로드까지 완료합니다.

## 실행 방법

```
/book-reviewer <책 제목>
```

### 예시
```
/book-reviewer 역행자
/book-reviewer 레버리지
/book-reviewer AI 전환 절대 공식
```

## API 정보

- **사이트**: https://doheelab.atlassian.net/
- **이메일**: doheelab@gmail.com
- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회
- **Jira API**: v3 (ADF 형식)
- **Confluence API**: v1 (Storage Format XHTML)

```bash
# 토큰 조회
cat ~/.claude/settings.json | jq -r '.mcpServers["jira-personal"].env.ATLASSIAN_API_TOKEN'
```

## 동작 절차

---

### Step 1: 책 정보 검색 (WebSearch)

WebSearch로 책 제목을 검색하여 다음 정보를 수집합니다:

- 저자, 출판사, 출간일, 페이지수, 카테고리
- 책 소개 (1-2문단)
- 목차 (챕터/파트 구조)

검색 쿼리 예시:
```
"<책 제목>" 책 저자 출판사 목차 소개
"<책 제목>" book review 목차 챕터
```

---

### Step 2: Jira 태스크 확인/생성 (필수)

`[목적] 독서` 에픽(TASK-10) 하위에서 해당 책의 태스크를 확인합니다.

**기존 태스크가 있으면** → 새로 생성하지 않고 기존 태스크를 사용합니다.
**기존 태스크가 없으면** → 새 태스크를 생성합니다.

#### 기존 태스크 확인 (python3 urllib)
```python
# TASK-10 하위 이슈에서 책 제목 검색
req = urllib.request.Request(
    'https://doheelab.atlassian.net/rest/api/3/search/jql?jql=parent=TASK-10+AND+summary~"<책제목>"',
    headers={'Authorization': f'Basic {auth}'}
)
```

#### 새 태스크 생성
```bash
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": {"key": "TASK"},
      "issuetype": {"name": "Task"},
      "parent": {"key": "TASK-10"},
      "summary": "<책 제목>",
      "description": <ADF 형식의 책 정보>
    }
  }'
```

#### Description ADF 구조
```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "책 정보"}]},
    {"type": "bulletList", "content": [
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "저자: ..."}]}]},
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "출판사: ..."}]}]},
      {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "출간일: ..."}]}]}
    ]},
    {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "책 소개"}]},
    {"type": "paragraph", "content": [{"type": "text", "text": "<소개글>"}]}
  ]
}
```

#### 활성 스프린트에 추가 (새 태스크 생성 시)
```bash
# 활성 스프린트 조회
curl -s -u "doheelab@gmail.com:<API_TOKEN>" \
  "https://doheelab.atlassian.net/rest/agile/1.0/board/1/sprint?state=active"

# 스프린트에 이슈 추가
curl -s -X POST -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/agile/1.0/sprint/<SPRINT_ID>/issue" \
  -d '{"issues":["<ISSUE_KEY>"]}'
```

---

### Step 3: 독서리뷰 마크다운 생성

목차 기반으로 챕터별 리뷰 템플릿을 생성하여 로컬에 저장합니다.

#### 저장 경로
```
/Users/dan.jung/Documents/code/dan-claude-skills/book-reviewer/notes/<책제목>.md
```

#### 템플릿

```markdown
# <책 제목>

## 책 정보
- **저자**: <저자>
- **출판사**: <출판사>
- **출간일**: <출간일>
- **카테고리**: <카테고리>
- **부제**: <부제>
- **Jira**: [TASK-XXX](https://doheelab.atlassian.net/browse/TASK-XXX)
- **Wiki**: [YYYY.MM.DD [독서리뷰] <책 제목>](<위키 URL>)

## 책 소개
<검색된 소개글 - 문장 단위로 줄바꿈하여 가독성 확보>

## 목차 & 독서리뷰

### 1장. <장 제목>
**핵심 내용**:
<충실한 요약 - 문장 단위로 줄바꿈>
<나열 항목은 불릿 리스트로 정리>

### 2장. <장 제목>
**핵심 내용**:
<충실한 요약 - 문장 단위로 줄바꿈>

(... 목차에 따라 반복)

## 읽은 후 정리
- **한 줄 요약**: <책의 핵심 메시지 한 문장>
- **핵심 교훈 3가지**:
  1. **<교훈 키워드>** - <설명>
  2. **<교훈 키워드>** - <설명>
  3. **<교훈 키워드>** - <설명>
- **실천 항목**:
  - [ ] <구체적 실천 항목 1>
  - [ ] <구체적 실천 항목 2>
  - [ ] <구체적 실천 항목 3>
```

#### 작성 규칙
- **핵심 내용**은 충실하게 작성 (책 요약이므로 내용이 풍부해야 함)
- 긴 문단은 문장 단위로 줄바꿈하여 가독성 확보
- 나열 항목(단계, 사례, 개념 등)은 불릿 리스트로 정리
- **읽은 후 정리**는 모두 채워서 작성

---

### Step 4: Confluence 위키 업로드 (필수)

**업로드 전 사용자에게 마크다운 내용을 확인받은 후 진행합니다.**

독서리뷰를 Confluence Storage Format (XHTML)으로 변환하여 위키 페이지를 생성합니다.

#### 위키 제목 규칙
```
YYYY.MM.DD [독서리뷰] <책 제목>
```
예시: `2026.02.13 [독서리뷰] 역행자`

#### 위키 분류체계
- **스페이스**: `~6167d0ad07ac3c00689b46b0` (개인 스페이스)
- **부모 페이지**: `독서리뷰` (ID: 40304716)
- 경로: `개요 > 목적 > 독서리뷰 > YYYY.MM.DD [독서리뷰] <책 제목>`

```bash
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/wiki/rest/api/content" \
  -d '{
    "type": "page",
    "title": "YYYY.MM.DD [독서리뷰] <책 제목>",
    "space": {"key": "~6167d0ad07ac3c00689b46b0"},
    "ancestors": [{"id": "40304716"}],
    "body": {
      "storage": {
        "value": "<XHTML 내용>",
        "representation": "storage"
      }
    }
  }'
```

#### 마크다운 → Confluence XHTML 변환 규칙

| 마크다운 | Confluence |
|---------|------------|
| `# 제목` | `<h1>제목</h1>` |
| `## 제목` | `<h2>제목</h2>` |
| `### 제목` | `<h3>제목</h3>` |
| `**굵게**` | `<strong>굵게</strong>` |
| `- 항목` | `<ul><li>항목</li></ul>` |
| `- [ ] TODO` | `<ac:task-list>...</ac:task-list>` |

---

### Step 5: Jira 코멘트로 위키 링크 추가

생성된 위키 페이지 URL을 Jira 태스크에 코멘트로 추가합니다.

```bash
curl -s -X POST \
  -u "doheelab@gmail.com:<API_TOKEN>" \
  -H "Content-Type: application/json" \
  "https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/comment" \
  -d '{
    "body": {
      "type": "doc",
      "version": 1,
      "content": [{"type": "paragraph", "content": [{"type": "text", "text": "독서리뷰 위키: <위키 URL>"}]}]
    }
  }'
```

---

### Step 6: 결과 반환

작업 완료 후 반드시 아래 정보를 반환합니다:

- **Jira 태스크**: https://doheelab.atlassian.net/browse/TASK-XXX
- **Confluence 위키**: https://doheelab.atlassian.net/wiki/spaces/~6167d0ad07ac3c00689b46b0/pages/XXXXX
- **로컬 파일**: `book-reviewer/notes/<책제목>.md`

---

## 에픽 규칙

독서 관련 태스크는 반드시 `[목적] 독서` 에픽(TASK-10) 하위에 생성합니다.

```
parent: {"key": "TASK-10"}
```

---

## 중요 지침

- **기존 Jira 태스크가 있으면 새로 생성하지 않고 기존 태스크 사용**
- **Confluence 업로드 전 사용자에게 마크다운 내용 확인받기**
- curl(python3 urllib)로 API 호출
- 작업 완료 후 결과 URL을 반드시 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- Confluence 페이지 업데이트 시 **버전 번호를 반드시 1 증가**
- 태스크 생성 후 현재 활성 스프린트의 TODO에 자동 추가
  - Board ID: 1, Agile API: `/rest/agile/1.0/board/1/sprint?state=active`
