# Claude Skills (개인)

개인 서비스 연동용 Claude Code 스킬 모음입니다.

## 스킬 목록

| 스킬 | 설명 | 연동 서비스 |
|------|------|------------|
| `dan-jira` | Jira 이슈 조회/생성/편집/코멘트/스프린트 | doheelab.atlassian.net |
| `dan-wiki` | Confluence 페이지 조회/생성/편집 | doheelab.atlassian.net |
| `dan-writing` | Jira + Confluence 통합 편집 | doheelab.atlassian.net |
| `dan-propose` | 프로필 이미지 분석 → 데이트 신청글 작성 | - |
| `dan-calendar` | Google Calendar 일정 조회/생성/수정/삭제 | Google Calendar API |

## 설치

```bash
# 저장소 클론
git clone https://github.com/doheelab-coder/claude-skills ~/Documents/code/claude-skills

# 심볼릭 링크 생성
for skill in ~/Documents/code/claude-skills/dan-*/; do
  name=$(basename "$skill")
  ln -sf "$skill" ~/.claude/skills/"$name"
done
```

## 구조

```
claude-skills/
├── dan-jira/          # 개인 Jira 관리
│   └── SKILL.md
├── dan-wiki/          # 개인 Confluence 관리
│   └── SKILL.md
├── dan-writing/       # Jira + Confluence 통합
│   └── SKILL.md
├── dan-propose/       # 데이트 신청글 작성
│   ├── SKILL.md
│   ├── .gitignore
│   ├── input/         # 프로필 이미지 (git 제외)
│   ├── output/        # 생성된 신청글 (git 제외)
│   └── archive/       # 처리 완료 이미지 (git 제외)
├── dan-calendar/      # Google Calendar 관리
│   └── SKILL.md
└── README.md
```
