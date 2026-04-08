# daily-scrum-plugin

Slack 데일리 스크럼 봇 스레드에 Linear 이슈 기반 자동 답글을 남기는 Claude Code 플러그인.

## 커맨드

- `/daily-scrum:setup` — 최초 설정 위자드 (MCP 연결 → 채널·봇 탐지 → 스케줄 생성)
- `/daily-scrum:modify` — 기존 스케줄 수정 (채널 변경, 시간 변경, 다중 채널 등)

## 구조

- `commands/setup.md` — 5단계 결정적 설정 플로우
- `commands/modify.md` — 스케줄 수정 플로우
- `prompts/schedule-template.md` — RemoteTrigger 프롬프트 템플릿

## 버전 관리

버전은 3곳에 동시 반영:
- `package.json` → `version`
- `.claude-plugin/plugin.json` → `version`
- `.claude-plugin/marketplace.json` → `version` (2곳: plugins[0].version + 루트 version)

## 플러그인 캐시 갱신

Claude Code 플러그인은 **2-layer 캐시** 구조. 버전 업데이트 후 반드시 양쪽 모두 클리어:

```bash
# 1. 언인스톨
claude plugins uninstall daily-scrum@daily-scrum

# 2. 양쪽 캐시 삭제 (핵심 — 하나만 지우면 갱신 안 됨)
rm -rf ~/.claude/plugins/marketplaces/daily-scrum   # 버전 목록 캐시
rm -rf ~/.claude/plugins/cache/daily-scrum          # 실제 코드 캐시

# 3. 재설치
claude plugins install daily-scrum@daily-scrum
```

`marketplaces/`만 지우면 코드가 안 바뀌고, `cache/`만 지우면 구버전으로 다시 설치됨.

## 의존성

런타임 코드 없음. 다음 MCP 커넥터 필요:

- Linear (claude_ai_Linear)
- Google Calendar (claude_ai_Google_Calendar)
- Slack (claude_ai_Slack)
