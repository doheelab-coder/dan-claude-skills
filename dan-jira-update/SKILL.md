---
name: dan-jira-update
description: Jira TASK 프로젝트 이슈 상태를 날짜 기준으로 자동 전환
---

# Dan Jira Update 스킬

TASK 프로젝트의 이슈 상태를 제목의 날짜 `(M/DD)` 기준으로 자동 전환합니다.

- **In Progress** 중 오늘 이전 날짜 → **Done**
- **To Do** 중 오늘 이전 날짜 → **Done**
- **To Do** 중 오늘 날짜 → **In Progress**
- **모든 상태**의 이슈를 날짜순으로 정렬 (같은 날짜 내 [관계] 에픽 우선)

## 실행 방법

```
/dan-jira-update
```

인자 없이 실행하면 오늘 날짜 기준으로 자동 처리합니다.

## API 정보

- **사이트**: https://doheelab.atlassian.net/
- **이메일**: doheelab@gmail.com
- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 호출**: 항상 curl(python3 urllib) 사용 (MCP 도구 사용하지 않음)

## 동작 절차

### 1단계: 오늘 날짜 확인

오늘 날짜를 `M/DD` 형식으로 준비합니다 (예: `2/14`, `12/3`).

- 앞에 0을 붙이지 않음: `2/14` (O), `02/14` (X)
- 현재 연도를 기준으로 날짜 비교

### 2단계: 활성 스프린트 이슈 전체 조회

활성 스프린트의 모든 이슈를 한 번에 조회합니다.

**JQL**: `project=TASK AND sprint in openSprints() AND assignee=currentUser() AND issuetype=Task ORDER BY status ASC, updated DESC`

#### curl로 조회 (기본)
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

### 3-1단계: 전체 이슈 날짜순 정렬

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

### 4단계: 사람 이슈에 Confluence 프로필 링크 추가

제목에 사람 이름이 포함된 이슈 (정규식 `\)\s*(.+?)님`)에 대해:

1. **Confluence 위키에서 프로필 페이지 검색**: `{이름} 프로필` 제목으로 검색
   - 스페이스 키: `~6167d0ad07ac3c00689b46b0`
   - 부모 페이지: 관계 (ID: 40370179)
2. **프로필이 존재하면**: 해당 이슈의 description에 위키 링크 추가
   - 기존 description 유지 + `프로필: {이름} 프로필 (Confluence)` 링크 행 추가
   - 이미 링크가 있으면 건너뜀
3. **프로필이 없으면**: 건너뜀

#### Confluence 인증
- `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN` + email (`doheelab@gmail.com`)
- Basic Auth (base64), python3 urllib 사용
- **반드시 메인 컨텍스트에서 직접 실행** (서브에이전트 위임 금지)

### 5단계: 결과 보고

전환 결과를 요약하여 출력합니다:

```
## 완료 처리 (In Progress → Done)
- TASK-101: (2/13) 서윤정님
- TASK-102: (2/12) 운동

## 완료 처리 (To Do → Done)
- TASK-155: (2/15) 달리기

## 진행 시작 (To Do → In Progress)
- TASK-105: (2/14) 김하나님
- TASK-106: (2/14) 독서

## 이슈 정렬 (날짜순, [관계] 우선)
### To Do
- TASK-154 (2/17) [관계] → TASK-135 (2/18) [관계] → TASK-142 (2/22)
### In Progress
- TASK-158 (2/16) [관계] → TASK-157 (2/16) [관계] → TASK-144 (2/16)
### Done
- TASK-137 (2/13) [관계] → TASK-147 (2/13) → ... → TASK-138 (2/15) [관계] → TASK-155 (2/15)

## 프로필 링크 추가
- TASK-105: 김하나님 → 김하나 프로필 (Confluence) 링크 추가

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
- **항상 curl(python3 urllib)로 API 호출** (MCP 도구 사용하지 않음)
- 전환 전 반드시 transitions 목록을 조회하여 올바른 ID 사용
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- 전환 대상이 없으면 "전환할 이슈가 없습니다"로 보고
