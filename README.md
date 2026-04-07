# daily-scrum-plugin

Slack 데일리 스크럼 봇 스레드에 Linear 이슈 기반 자동 답글을 남기는 Claude Code 플러그인.

매일 아침 정해진 시각에:
1. Google Calendar로 휴일/휴가 여부를 확인하고
2. Linear에서 오늘 할 일(Todo/In Progress)과 어제 한 일(Done)을 수집하여
3. Slack 데일리 스크럼 봇의 스레드에 자동으로 답글을 남깁니다.

## 설치

Claude Code 설정에서 플러그인을 추가합니다:

```bash
# 로컬 경로로 추가
claude plugin add /path/to/daily-scrum-plugin
```

또는 `~/.claude/settings.json`에 직접 추가:

```json
{
  "plugins": [
    "/path/to/daily-scrum-plugin"
  ]
}
```

## 사용법

### 최초 설정

```
/daily-scrum:setup
```

대화형 위자드가 5단계로 안내합니다:

1. **MCP 연결 확인** — Linear, Google Calendar, Slack 커넥터 연결
2. **Slack 채널·봇 탐지** — 데일리 스크럼 채널과 봇 자동 감지
3. **Linear 범위 설정** — 이슈를 가져올 범위 선택
4. **캘린더 휴일 패턴** — 휴일/휴가 판단 기준 설정
5. **스케줄 생성** — RemoteTrigger 클라우드 스케줄 등록

### 설정 수정

```
/daily-scrum:modify
```

기존 스케줄의 설정을 변경합니다:
- 채널 변경 / 추가 (다중 채널 지원)
- 답글 시각 변경
- Linear 범위 변경
- 휴일 기준 변경
- 스케줄 일시 중지 / 재개 / 삭제

## 필요한 MCP 커넥터

다음 3개 서비스가 Claude에 연결되어 있어야 합니다:

| 서비스 | 용도 | 연결 방법 |
|--------|------|-----------|
| Linear | 이슈 조회 (Todo/Done) | Claude Desktop → Settings → Integrations |
| Google Calendar | 휴일/휴가 판단 | 동일 |
| Slack | 채널 조회, 답글 전송 | 동일 |

연결 상태는 `/daily-scrum:setup` 실행 시 자동으로 확인됩니다.

## 메시지 예시

```
성호
📋 *오늘 할 일*
• PROJ-123 로그인 페이지 리디자인
• PROJ-456 API 응답 캐싱 구현

✅ *어제 한 일*
• PROJ-789 사이드바 버그 수정
• PROJ-012 테스트 커버리지 개선
```

- 휴일/휴가일에는 자동 스킵
- 월요일에는 금~일 완료 이슈가 "지난주 한 일"로 표시
- 봇이 스레드를 올리지 않은 날에는 자동 스킵
