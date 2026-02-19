# Claude Skills (개인)

개인 서비스 연동용 Claude Code 스킬 모음입니다.

## 스킬 목록

| 스킬 | 설명 | 연동 서비스 |
|------|------|------------|
| `dan-jira` | Jira 이슈 조회/생성/편집/코멘트/스프린트 | doheelab.atlassian.net |
| `dan-wiki` | Confluence 페이지 조회/생성/편집 | doheelab.atlassian.net |
| `dan-writing` | Jira + Confluence 통합 편집 | doheelab.atlassian.net |
| `dan-calendar` | Google Calendar 일정 조회/생성/수정/삭제 | Google Calendar API |
| `dan-jira-update` | Jira 이슈 상태 자동 전환 + Calendar 일정 검증/수정 | doheelab.atlassian.net, Google Calendar API |
| `dan-tech-reviewer` | 최신 기술 동향/뉴스 검색 → 리포트 작성 | doheelab.atlassian.net |
| `dan-youtube-reviewer` | YouTube 영상 자막 추출 → 내용 요약 리뷰 | doheelab.atlassian.net, YouTube |

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
├── dan-calendar/      # Google Calendar 관리
│   └── SKILL.md
├── dan-jira-update/   # Jira 상태 전환 + Calendar 검증
│   └── SKILL.md
├── dan-tech-reviewer/   # 기술 동향 리포트
│   ├── SKILL.md
│   ├── .gitignore
│   └── output/        # 리포트 (git 제외)
├── dan-youtube-reviewer/  # YouTube 영상 리뷰
│   ├── SKILL.md
│   ├── .gitignore
│   └── output/        # 리포트 (git 제외)
└── README.md
```
