# daily-scrum-plugin

Slack 데일리 스크럼 봇 스레드에 Linear 이슈 기반 자동 답글을 남기는 Claude Code 플러그인.

## 커맨드

- `/daily-scrum:setup` — 최초 설정 위자드 (MCP 연결 → 채널·봇 탐지 → 스케줄 생성)
- `/daily-scrum:modify` — 기존 스케줄 수정 (채널 변경, 시간 변경, 다중 채널 등)

## 구조

- `commands/setup.md` — 5단계 결정적 설정 플로우
- `commands/modify.md` — 스케줄 수정 플로우
- `prompts/schedule-template.md` — RemoteTrigger 프롬프트 템플릿

## 의존성

런타임 코드 없음. 다음 MCP 커넥터 필요:
- Linear (claude_ai_Linear)
- Google Calendar (claude_ai_Google_Calendar)
- Slack (claude_ai_Slack)
