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
| `dan-jira-update` | Jira 이슈 상태를 날짜 기준으로 자동 전환 | doheelab.atlassian.net |
| `dan-person-analysis` | 프로필/카카오톡 대화 분석 → 인물 정보 정리 | - |
| `dan-tech-reviewer` | 최신 기술 동향/뉴스 검색 → 리포트 작성 | doheelab.atlassian.net |

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
├── dan-jira-update/   # Jira 이슈 상태 자동 전환
│   └── SKILL.md
├── dan-person-analysis/  # 인물 정보 분석
│   ├── SKILL.md
│   ├── .gitignore
│   ├── input/         # 사람별 디렉토리 (git 제외)
│   └── output/        # 분석 결과 (git 제외)
├── dan-tech-reviewer/   # 기술 동향 리포트
│   ├── SKILL.md
│   ├── .gitignore
│   └── output/        # 리포트 (git 제외)
└── README.md
```
