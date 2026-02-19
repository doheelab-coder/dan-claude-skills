---
name: dan-youtube-reviewer
description: YouTube 영상의 자막을 추출하여 내용을 요약하고 Jira/Confluence에 등록
---

# Dan YouTube Reviewer 스킬

YouTube 링크를 입력하면 자막(transcript)을 추출하고, 영상 내용을 구조화된 리뷰로 정리한 뒤, 로컬 마크다운 저장 + Jira 태스크 + Confluence 위키 업로드까지 자동 완료합니다.

## 실행 방법

```
/dan-youtube-reviewer <YouTube URL>
```

### 예시
```
/dan-youtube-reviewer https://www.youtube.com/watch?v=dQw4w9WgXcQ
/dan-youtube-reviewer https://youtu.be/dQw4w9WgXcQ
```

## 사전 요구사항

`youtube-transcript-api` 패키지가 필요합니다. 없으면 자동 설치합니다.

```bash
pip install youtube-transcript-api
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

### Step 1: YouTube URL 파싱 및 영상 정보 수집

사용자 입력에서 YouTube Video ID를 추출합니다.

```python
import re

def extract_video_id(url):
    patterns = [
        r'(?:youtube\.com/watch\?v=|youtu\.be/|youtube\.com/embed/)([a-zA-Z0-9_-]{11})',
    ]
    for p in patterns:
        m = re.search(p, url)
        if m:
            return m.group(1)
    return None
```

WebFetch로 YouTube 페이지에서 영상 메타데이터를 수집합니다:
- 영상 제목
- 채널명
- 업로드 날짜
- 영상 길이
- 설명(description)

```python
# WebFetch로 YouTube 페이지 메타데이터 추출
# URL: https://www.youtube.com/watch?v=<VIDEO_ID>
# prompt: "영상 제목, 채널명, 업로드 날짜, 영상 길이, 설명을 추출해줘"
```

---

### Step 2: 자막(Transcript) 추출

`youtube-transcript-api`로 자막을 추출합니다. 한국어 → 영어 → 자동생성 순으로 시도합니다.

```python
import subprocess, sys

# 패키지 확인 및 설치
try:
    from youtube_transcript_api import YouTubeTranscriptApi
except ImportError:
    subprocess.check_call([sys.executable, '-m', 'pip', 'install', 'youtube-transcript-api'])
    from youtube_transcript_api import YouTubeTranscriptApi

# 자막 추출 (한국어 우선, 영어 fallback)
try:
    fetcher = YouTubeTranscriptApi()
    transcript = fetcher.fetch(video_id, languages=['ko', 'en'])
except Exception:
    # 자동 생성 자막 시도
    transcript_list = YouTubeTranscriptApi().list(video_id)
    transcript = transcript_list.find_transcript(['ko', 'en']).fetch()

# 전체 텍스트로 결합
full_text = '\n'.join([entry['text'] for entry in transcript])
```

**자막이 없는 경우**: WebFetch로 영상 페이지의 설명(description)만 활용하여 제한적 요약을 수행하고, 자막 미제공 사실을 명시합니다.

---

### Step 3: 내용 분석 및 요약

추출된 자막 텍스트를 분석하여 다음을 정리합니다:

1. **영상 개요**: 전체 내용 2-3문단 요약
2. **핵심 주제**: 영상에서 다루는 주요 토픽 3-7개
3. **타임라인 요약**: 자막의 타임스탬프 기반으로 구간별 내용 정리
4. **핵심 인사이트**: 영상에서 얻을 수 있는 핵심 교훈/정보
5. **인용구**: 인상적인 발언 2-3개 (타임스탬프 포함)

---

### Step 4: 마크다운 리포트 생성

#### 저장 경로
```
/Users/dan.jung/Documents/code/dan-claude-skills/dan-youtube-reviewer/output/<영상제목>_<YYYY-MM-DD>.md
```
- 파일명의 `<영상제목>`은 공백을 `_`로, 특수문자 제거 (예: `LLM_에이전트_소개_2026-02-20.md`)

#### 템플릿

```markdown
# <영상 제목>

> 작성일: YYYY-MM-DD
> Jira: [TASK-XXX](https://doheelab.atlassian.net/browse/TASK-XXX)
> Wiki: [YYYY.MM.DD [영상리뷰] <영상 제목>](<위키 URL>)

## 영상 정보
- **채널**: <채널명>
- **업로드일**: <업로드 날짜>
- **영상 길이**: <길이>
- **링크**: <YouTube URL>
- **자막 언어**: <ko/en/auto>

## 개요
<영상 전체 내용을 2-3문단으로 요약>

## 핵심 주제

### 1. <주제 키워드>
<설명 - 문장 단위로 줄바꿈하여 가독성 확보>

### 2. <주제 키워드>
<설명>

### 3. <주제 키워드>
<설명>

(주제 수는 내용에 따라 3~7개)

## 타임라인 요약

| 시간 | 내용 |
|------|------|
| 0:00 | <구간 내용 요약> |
| 5:30 | <구간 내용 요약> |
| ... | ... |

## 핵심 인사이트
- <인사이트 1>
- <인사이트 2>
- <인사이트 3>

## 인용구
> "<인상적인 발언 1>" (MM:SS)

> "<인상적인 발언 2>" (MM:SS)

## 출처
- [<영상 제목>](<YouTube URL>)
```

#### 작성 규칙
- **개요**는 비전문가도 이해할 수 있도록 명확하게 작성
- **핵심 주제**는 중요도순으로 정리, 각 주제에 구체적 내용 포함
- **타임라인**은 주요 전환점 기준 5-10개 구간으로 정리
- 긴 문단은 문장 단위로 줄바꿈
- 자막 언어가 영어인 경우 한국어로 번역하여 작성
- **출처** 섹션에 원본 YouTube URL 기재

---

### Step 5: Jira 태스크 확인/생성

`[목적] 스터디` 에픽(TASK-9) 하위에서 해당 영상의 태스크를 확인합니다.

**기존 태스크가 있으면** → 새로 생성하지 않고 기존 태스크를 사용합니다.
**기존 태스크가 없으면** → 새 태스크를 생성합니다.

#### 기존 태스크 확인 (python3 urllib)
```python
req = urllib.request.Request(
    'https://doheelab.atlassian.net/rest/api/3/search/jql?jql=parent=TASK-9+AND+summary~"<영상제목 키워드>"',
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
        "summary": "[영상리뷰] <영상 제목>",
        "description": {
            "type": "doc", "version": 1,
            "content": [
                {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "영상 정보"}]},
                {"type": "bulletList", "content": [
                    {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "채널: <채널명>"}]}]},
                    {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "길이: <영상 길이>"}]}]},
                    {"type": "listItem", "content": [{"type": "paragraph", "content": [
                        {"type": "text", "text": "링크: "},
                        {"type": "text", "text": "<YouTube URL>", "marks": [{"type": "link", "attrs": {"href": "<YouTube URL>"}}]}
                    ]}]}
                ]},
                {"type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "개요"}]},
                {"type": "paragraph", "content": [{"type": "text", "text": "<영상 개요 요약>"}]}
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
YYYY.MM.DD [영상리뷰] <영상 제목>
```
예시: `2026.02.20 [영상리뷰] LLM 에이전트 소개`

#### 위키 분류체계
- **스페이스**: `~6167d0ad07ac3c00689b46b0` (개인 스페이스)
- **부모 페이지**: `스터디` (CQL 검색으로 ID 조회)
- 경로: `개요 > 목적 > 스터디 > YYYY.MM.DD [영상리뷰] <영상 제목>`

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
    "title": "YYYY.MM.DD [영상리뷰] <영상 제목>",
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
            {"type": "text", "text": "영상리뷰 위키: "},
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
- **로컬 파일**: `dan-youtube-reviewer/output/<영상제목>_<날짜>.md`
- **리포트 요약**: 핵심 주제 3줄 요약

---

## 에픽 규칙

영상 리뷰 태스크는 반드시 `[목적] 스터디` 에픽(TASK-9) 하위에 생성합니다.

```
parent: {"key": "TASK-9"}
```

---

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- curl(python3 urllib)로 API 호출
- 자막 추출 실패 시 WebFetch로 영상 설명만이라도 활용
- 영어 자막은 한국어로 번역하여 리포트 작성
- 확인되지 않은 정보는 추측으로 표기하지 않음
- 작업 완료 후 결과 URL을 반드시 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- **로컬 마크다운을 수정할 때는 반드시 Confluence 위키 페이지도 함께 업데이트할 것**
- Confluence 페이지 업데이트 시 **버전 번호를 반드시 1 증가**
- 태스크 생성 후 현재 활성 스프린트에 자동 추가 + **In Progress 전환**
  - Board ID: 1, Agile API: `/rest/agile/1.0/board/1/sprint?state=active`
