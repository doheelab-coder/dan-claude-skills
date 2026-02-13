---
name: asset-update
description: Notion 재무제표 데이터를 Confluence 위키에 자동 동기화
---

# Asset Update 스킬

Notion에서 관리하는 재무제표 테이블 데이터를 가져와서 Confluence "재무제표" 위키 페이지의 표를 업데이트합니다.

## 실행 방법

```
/asset-update
```

## API 정보

- **사이트**: https://doheelab.atlassian.net/
- **이메일**: doheelab@gmail.com
- **인증 방식**: Atlassian Cloud Basic Auth (email:api_token)
- **API 토큰**: `~/.claude/settings.json`의 `mcpServers.jira-personal.env.ATLASSIAN_API_TOKEN`에서 조회
- **Confluence API**: v1 (Storage Format XHTML)

```bash
# 토큰 조회
cat ~/.claude/settings.json | jq -r '.mcpServers["jira-personal"].env.ATLASSIAN_API_TOKEN'
```

## 동작 절차

---

### Step 1: Notion 데이터 조회

Notion 공개 API로 재무제표 테이블 데이터를 조회합니다.

```python
import urllib.request, json

data = json.dumps({
    "page": {"id": "d37c878a-3c0c-4c1d-af70-3af489a06f4e"},
    "limit": 100,
    "cursor": {"stack": []},
    "chunkNumber": 0,
    "verticalColumns": False
}).encode()

req = urllib.request.Request(
    'https://www.notion.so/api/v3/loadCachedPageChunk',
    data=data,
    headers={'Content-Type': 'application/json'}
)
result = json.loads(urllib.request.urlopen(req).read())
blocks = result.get('recordMap', {}).get('block', {})
```

#### Notion 컬럼 매핑

| Notion Key | 필드명 | 설명 |
|------------|--------|------|
| `WIX<` | date | 일자 |
| `F;qt` | asset | 총액 (만원) |
| `geDi` | target | 목표금액 (만원) |
| `DYNQ` | surplus | 초과금액 (만원) |
| `SOhC` | profit | 이익 (만원) |
| `V@WD` | ahead | 속도 (개월) |
| `V@;P` | expense | 소비 |

#### 필드 계산 공식

기본 파라미터:
- **저축액**: 월 400만원
- **목표 수익률**: 연 5.5%
- **월 수익률**: `1.055^(1/12) - 1` ≈ 0.4468%

| 필드 | 계산 공식 | 설명 |
|------|-----------|------|
| target | `previous_target * 1.055^(1/12) + 400` | 전월 목표금액에 월 수익률 적용 + 저축액 |
| surplus | `asset - target` | 실제 총액 - 목표금액 |
| profit | `target * (1.055^(1/12) - 1)` | 목표금액 기준 월 이익 (만원, 반올림) |
| ahead | 해당월 asset이 몇 개월 앞선 target을 초과했는지 계산 | 예: +3이면 3개월 앞선 목표를 달성 |
| expense | 해당월의 주요 지출 메모 | 텍스트 (예: "아반떼 1000만원") |

#### 데이터 추출

```python
for bid, bdata in blocks.items():
    val = bdata.get('value', {})
    if val.get('type') == 'table_row':
        props = val.get('properties', {})
        date = props.get('WIX<', [['']])[0][0].strip()
        asset = props.get('F;qt', [['']])[0][0].strip()
        target = props.get('geDi', [['']])[0][0].strip()
        surplus = props.get('DYNQ', [['']])[0][0].strip()
        profit = props.get('SOhC', [['']])[0][0].strip()
        ahead = props.get('V@WD', [['']])[0][0].strip()
        expense = props.get('V@;P', [['']])[0][0].strip() if 'V@;P' in props else ''
```

---

### Step 2: Confluence 현재 데이터 조회

Confluence 재무제표 페이지의 현재 표 데이터를 조회합니다.

#### 페이지 정보
- **페이지 ID**: 6062082
- **스페이스**: `~6167d0ad07ac3c00689b46b0` (개인 스페이스)
- **페이지 URL**: https://doheelab.atlassian.net/wiki/spaces/~6167d0ad07ac3c00689b46b0/pages/6062082

```python
import base64

token = json.load(open('/Users/dan.jung/.claude/settings.json'))['mcpServers']['jira-personal']['env']['ATLASSIAN_API_TOKEN']
auth = base64.b64encode(f'doheelab@gmail.com:{token}'.encode()).decode()

req = urllib.request.Request(
    'https://doheelab.atlassian.net/wiki/rest/api/content/6062082?expand=version,body.storage',
    headers={'Authorization': f'Basic {auth}'}
)
resp = json.loads(urllib.request.urlopen(req).read())
version = resp['version']['number']
body = resp['body']['storage']['value']
```

#### 표 데이터 파싱 (HTML에서 추출)

```python
import re

rows = re.findall(r'<tr[^>]*>(.*?)</tr>', body, re.DOTALL)
for row in rows:
    cells = re.findall(r'<t[hd][^>]*>(.*?)</t[hd]>', row, re.DOTALL)
    cell_texts = [re.sub(r'<[^>]+>', '', cell).strip() for cell in cells]
```

---

### Step 3: 차이점 식별 및 표시

Notion과 Confluence 데이터를 비교하여 변경사항을 사용자에게 표시합니다.

비교 항목:
- 새로 채워진 실적 데이터 (빈 셀 → 실제 값)
- 변경된 목표금액/이익 수치
- 새로 추가된 소비 메모

**표시 형식:**
```
| 월 | 총액 | 목표금액 | 초과금액 | 이익 | 소비 |
|---|---|---|---|---|---|
| 2025.12 | → 45922만 | 38969 | → +6953 | 173→174 | |
```

---

### Step 4: Confluence 표 업데이트

HTML body에서 해당 셀 값을 targeted replacement로 업데이트합니다.

**주의사항:**
- HTML body가 ~165KB로 매우 큼 → 전체 출력 금지, targeted replacement 사용
- 각 행은 날짜 셀(`<strong>2025.12</strong>` 등)로 식별
- 빈 셀은 `<p></p>` 또는 빈 `<td>` 패턴
- 버전 번호를 반드시 +1로 설정

```python
# 업데이트 예시
body = body.replace(old_value, new_value)

data = json.dumps({
    "type": "page",
    "title": "재무제표",
    "version": {"number": version + 1},
    "body": {
        "storage": {
            "value": body,
            "representation": "storage"
        }
    }
}).encode()

req = urllib.request.Request(
    'https://doheelab.atlassian.net/wiki/rest/api/content/6062082',
    data=data,
    headers={
        'Authorization': f'Basic {auth}',
        'Content-Type': 'application/json'
    },
    method='PUT'
)
```

---

### Step 5: 결과 반환

작업 완료 후 반드시 아래 정보를 반환합니다:

- **변경 요약**: 업데이트된 행/셀 목록
- **Confluence 위키**: https://doheelab.atlassian.net/wiki/spaces/~6167d0ad07ac3c00689b46b0/pages/6062082

---

## 표 구조

### 메인 표 (2025.01~)

| 일자 | 총액 | 목표금액 | 초과금액 | 이익 | 소비 |
|------|------|----------|----------|------|------|
| 2025.01 | 32809만 | 32800만 | 0 | - | 아반떼 1000만원 |
| ... | ... | ... | ... | ... | ... |

### 2024 히스토리 표

| 일자 | 총액 | 목표금액 | 초과금액 | 소비 |
|------|------|----------|----------|------|
| 2024.02.09 | 25511만 | 25511만 | | |
| ... | ... | ... | ... | ... |

---

## 중요 지침

- **진행 여부를 묻지 않고 끝까지 완료할 것**
- curl(python3 urllib)로 API 호출
- 작업 완료 후 결과 URL을 반드시 반환
- 오류 발생 시 원인을 파악하고 가능한 대안을 시도
- Confluence 페이지 업데이트 시 **버전 번호를 반드시 1 증가**
- HTML body가 매우 크므로 **전체 출력 금지, targeted replacement 사용**
