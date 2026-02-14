---
name: dan-tech-reviewer
description: 최신 기술 동향과 뉴스를 검색하여 리포트 작성 (로컬 + Jira + Confluence)
---

# Dan Tech Reviewer 스킬

기술 주제를 입력하면 WebSearch로 최신 동향, 뉴스, 블로그, 논문을 검색하고 구조화된 리포트를 생성합니다.
로컬 마크다운 저장 + Jira 태스크 (진행중) + Confluence 위키 페이지까지 자동 완료합니다.

## 실행 방법

```
/dan-tech-reviewer <기술 주제>
```

### 예시
```
/dan-tech-reviewer LLM 에이전트
/dan-tech-reviewer RAG 최신 동향
/dan-tech-reviewer 광고 추천 시스템 트렌드
/dan-tech-reviewer Rust vs Go 2026
/dan-tech-reviewer MCP (Model Context Protocol)
```

## API 정보

- **사이트**: https://doheelab.atlassian.net/
- **이메일**: doheelab@gmail.com
- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회
- **Jira API**: v3 (ADF 형식)
- **Confluence API**: v1 (Storage Format XHTML)

### 토큰 로드 패턴

```python
import json, urllib.request, base64
d = json.load(open('/Users/dan.jung/.claude/settings.json'))
token = d['mcpServers']['jira-personal']['env']['ATLASSIAN_API_TOKEN']
creds = base64.b64encode(f'doheelab@gmail.com:{token}'.encode()).decode()
# req.add_header('Authorization', f'Basic {creds}')
```

## 동작 절차

---

### Step 1: 주제 파싱

사용자 입력에서 기술 주제를 추출합니다. 모호한 경우 검색 키워드를 구체화합니다.

---

### Step 2: 웹 검색 (WebSearch 3~5회)

다양한 각도로 검색하여 폭넓은 정보를 수집합니다.

#### 검색 쿼리 전략
```
1. "<주제> 2026 최신 동향" (한국어 동향)
2. "<topic> 2026 latest trends" (영문 트렌드)
3. "<topic> news 2026" (최신 뉴스)
4. "<주제> 기술 블로그 비교 분석" (심층 분석)
5. "<topic> research paper 2026" (논문/연구, 필요시)
```

#### 주요 수집 정보
- 기술 개요 및 현재 상태
- 최신 뉴스 및 발표 (최근 3개월 이내)
- 주요 기업/프로젝트 동향
- 기술 비교 및 장단점
- 향후 전망

---

### Step 3: 핵심 아티클 읽기 (WebFetch)

검색 결과에서 가장 관련성 높은 아티클 3~5개를 WebFetch로 읽고 핵심 내용을 추출합니다.

---

### Step 4: 마크다운 리포트 생성

#### 저장 경로
```
/Users/dan.jung/Documents/code/dan-claude-skills/dan-tech-reviewer/output/<주제>_<YYYY-MM-DD>.md
```
- 파일명의 `<주제>`는 공백을 `_`로 치환 (예: `LLM_에이전트_2026-02-14.md`)

#### 템플릿

```markdown
# <주제> 기술 동향 리포트

> 작성일: YYYY-MM-DD
> Jira: [TASK-XXX](https://doheelab.atlassian.net/browse/TASK-XXX)
> Wiki: [YYYY.MM.DD [기술리뷰] <주제>](<위키 URL>)

## 개요
<기술의 정의, 배경, 현재 위치를 2-3문단으로 요약>

## 주요 동향

### 1. <동향 키워드>
<설명 - 문장 단위로 줄바꿈하여 가독성 확보>

### 2. <동향 키워드>
<설명>

### 3. <동향 키워드>
<설명>

(동향 수는 내용에 따라 3~5개)

## 핵심 뉴스 & 아티클

### <아티클 제목 1>
- **출처**: <매체/블로그명>
- **날짜**: YYYY-MM-DD (확인 가능한 경우)
- **요약**: <3-5줄 요약>
- **링크**: <URL>

### <아티클 제목 2>
- **출처**: <매체/블로그명>
- **날짜**: YYYY-MM-DD
- **요약**: <3-5줄 요약>
- **링크**: <URL>

(3~5개 아티클)

## 주요 플레이어 & 프로젝트

| 이름 | 설명 | 링크 |
|------|------|------|
| <기업/프로젝트> | <한줄 설명> | <URL> |

## 시사점
- <핵심 인사이트 1>
- <핵심 인사이트 2>
- <핵심 인사이트 3>

## 출처
- [제목](URL)
- [제목](URL)
(검색 및 참조한 모든 URL)
```

#### 작성 규칙
- **개요**는 비전문가도 이해할 수 있도록 명확하게 작성
- **주요 동향**은 최신순으로 정리, 각 동향에 구체적 사례 포함
- **핵심 뉴스**는 최근 3개월 이내 자료 우선
- 긴 문단은 문장 단위로 줄바꿈
- 한국어 자료와 영문 자료를 균형있게 포함
- **출처** 섹션에 참조한 모든 URL을 빠짐없이 기재

---

### Step 5: Jira 태스크 확인/생성

`[목적] 스터디` 에픽(TASK-9) 하위에서 해당 주제의 태스크를 확인합니다.

**기존 태스크가 있으면** → 새로 생성하지 않고 기존 태스크를 사용합니다.
**기존 태스크가 없으면** → 새 태스크를 생성합니다.

#### 기존 태스크 확인 (python3 urllib)
```python
req = urllib.request.Request(
    'https://doheelab.atlassian.net/rest/api/3/search/jql?jql=parent=TASK-9+AND+summary~"<주제>"',
    headers={'Authorization': f'Basic {auth}'}
)
```

#### 새 태스크 생성
```python
data = json.dumps({
    "fields": {
        "project": {"key": "TASK"},
        "issuetype": {"name": "Task"},
        "parent": {"key": "TASK-9"},
        "summary": "[기술리뷰] <주제>",
        "description": {
            "type": "doc", "version": 1,
            "content": [
                {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "기술 동향 리포트"}]},
                {"type": "paragraph", "content": [{"type": "text", "text": "<개요 요약>"}]}
            ]
        }
    }
}).encode()
req = urllib.request.Request(
    'https://doheelab.atlassian.net/rest/api/3/issue',
    data=data, method='POST',
    headers={'Authorization': f'Basic {auth}', 'Content-Type': 'application/json'}
)
```

#### 활성 스프린트에 추가 + In Progress 전환
```python
# 1. 활성 스프린트 조회
req = urllib.request.Request(
    'https://doheelab.atlassian.net/rest/agile/1.0/board/1/sprint?state=active',
    headers={'Authorization': f'Basic {auth}'}
)

# 2. 스프린트에 이슈 추가
data = json.dumps({"issues": ["<ISSUE_KEY>"]}).encode()
req = urllib.request.Request(
    f'https://doheelab.atlassian.net/rest/agile/1.0/sprint/<SPRINT_ID>/issue',
    data=data, method='POST',
    headers={'Authorization': f'Basic {auth}', 'Content-Type': 'application/json'}
)

# 3. In Progress 상태로 전환
# 먼저 전환 가능한 상태 조회
req = urllib.request.Request(
    f'https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/transitions',
    headers={'Authorization': f'Basic {auth}'}
)
# "In Progress" transition ID를 찾아서 전환
data = json.dumps({"transition": {"id": "<IN_PROGRESS_TRANSITION_ID>"}}).encode()
req = urllib.request.Request(
    f'https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/transitions',
    data=data, method='POST',
    headers={'Authorization': f'Basic {auth}', 'Content-Type': 'application/json'}
)
```

---

### Step 6: Confluence 위키 페이지 생성

리포트를 Confluence Storage Format (XHTML)으로 변환하여 위키 페이지를 생성합니다.

#### 위키 제목 규칙
```
YYYY.MM.DD [기술리뷰] <주제>
```
예시: `2026.02.14 [기술리뷰] LLM 에이전트`

#### 위키 분류체계
- **스페이스**: `~6167d0ad07ac3c00689b46b0` (개인 스페이스)
- **부모 페이지**: `스터디` (ID 조회 필요 → CQL 검색)
- 경로: `개요 > 목적 > 스터디 > YYYY.MM.DD [기술리뷰] <주제>`

```python
# 부모 페이지 검색 (스터디)
import urllib.parse
cql = urllib.parse.quote('title = "스터디" AND space.key = "~6167d0ad07ac3c00689b46b0"')
req = urllib.request.Request(
    f'https://doheelab.atlassian.net/wiki/rest/api/content/search?cql={cql}',
    headers={'Authorization': f'Basic {auth}'}
)

# 페이지 생성
data = json.dumps({
    "type": "page",
    "title": "YYYY.MM.DD [기술리뷰] <주제>",
    "space": {"key": "~6167d0ad07ac3c00689b46b0"},
    "ancestors": [{"id": "<PARENT_PAGE_ID>"}],
    "body": {
        "storage": {
            "value": "<XHTML 내용>",
            "representation": "storage"
        }
    }
}).encode()
req = urllib.request.Request(
    'https://doheelab.atlassian.net/wiki/rest/api/content',
    data=data, method='POST',
    headers={'Authorization': f'Basic {auth}', 'Content-Type': 'application/json'}
)
```

#### 마크다운 → Confluence XHTML 변환 규칙

| 마크다운 | Confluence |
|---------|------------|
| `# 제목` | `<h1>제목</h1>` |
| `## 제목` | `<h2>제목</h2>` |
| `### 제목` | `<h3>제목</h3>` |
| `**굵게**` | `<strong>굵게</strong>` |
| `- 항목` | `<ul><li>항목</li></ul>` |
| `[텍스트](URL)` | `<a href="URL">텍스트</a>` |
| 테이블 | `<table><tbody><tr><th>...</th></tr><tr><td>...</td></tr></tbody></table>` |
| `> 인용` | `<blockquote><p>인용</p></blockquote>` |

---

### Step 7: Jira 코멘트로 위키 링크 추가

생성된 위키 페이지 URL을 Jira 태스크에 코멘트로 추가합니다.

```python
data = json.dumps({
    "body": {
        "type": "doc", "version": 1,
        "content": [{"type": "paragraph", "content": [
            {"type": "text", "text": "기술리뷰 위키: "},
            {"type": "text", "text": "<위키 URL>", "marks": [{"type": "link", "attrs": {"href": "<위키 URL>"}}]}
        ]}]
    }
}).encode()
req = urllib.request.Request(
    f'https://doheelab.atlassian.net/rest/api/3/issue/<ISSUE_KEY>/comment',
    data=data, method='POST',
    headers={'Authorization': f'Basic {auth}', 'Content-Type': 'application/json'}
)
```

---

### Step 8: 로컬 마크다운에 링크 업데이트

Jira 태스크 키와 Confluence 위키 URL을 로컬 마크다운 파일의 헤더에 반영합니다.

---

### Step 9: 결과 반환

작업 완료 후 반드시 아래 정보를 반환합니다:

- **Jira 태스크**: https://doheelab.atlassian.net/browse/TASK-XXX
- **Confluence 위키**: https://doheelab.atlassian.net/wiki/spaces/~6167d0ad07ac3c00689b46b0/pages/XXXXX
- **로컬 파일**: `dan-tech-reviewer/output/<주제>_<날짜>.md`
- **리포트 요약**: 주요 동향 3줄 요약

---

## 에픽 규칙

기술 리뷰 태스크는 반드시 `[목적] 스터디` 에픽(TASK-9) 하위에 생성합니다.

```
parent: {"key": "TASK-9"}
```

---

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- curl(python3 urllib)로 API 호출
- WebSearch 결과만으로 불충분하면 WebFetch로 아티클 본문을 읽어 보충
- 검색 결과가 부실하면 키워드를 변형하여 재검색 (영문 ↔ 한국어)
- 오래된 정보(1년 이상)는 명시적으로 날짜를 표기하고 최신 정보와 구분
- 확인되지 않은 정보는 추측으로 표기하지 않음
- 작업 완료 후 결과 URL을 반드시 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- **로컬 마크다운을 수정할 때는 반드시 Confluence 위키 페이지도 함께 업데이트할 것**
- Confluence 페이지 업데이트 시 **버전 번호를 반드시 1 증가**
- 태스크 생성 후 현재 활성 스프린트에 자동 추가 + **In Progress 전환**
  - Board ID: 1, Agile API: `/rest/agile/1.0/board/1/sprint?state=active`
