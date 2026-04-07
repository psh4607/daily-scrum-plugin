# 데일리 스크럼 자동 답글

매일 아침 Slack 데일리 스크럼 봇 스레드에 Linear 이슈 기반 답글을 남긴다.

## 설정 정보

- 유저: {{USER_NAME}}
- 채널: {{CHANNEL_NAMES}}
- 채널별 봇 정보: {{CHANNELS_CONFIG}}
- 봇 게시 시각: {{BOT_TIME}}
- 답글 시각: {{REPLY_TIME}}
- Linear 범위: {{LINEAR_SCOPE_DESC}}
- 휴가 판단 키워드: {{HOLIDAY_KEYWORDS}}

## 실행 절차

아래 4단계를 순서대로 수행한다. 각 단계의 조건부 종료 로직을 반드시 따른다.

---

### 1단계: 휴일/휴가 확인

`gcal_list_events`로 오늘 하루의 캘린더 이벤트를 조회한다.

- 기간: 오늘 00:00 ~ 오늘 23:59 (로컬 타임존)
- 조회 대상: 기본 캘린더

조회된 이벤트 중 **종일 이벤트**의 제목에 다음 키워드가 포함되어 있는지 확인한다:
`{{HOLIDAY_KEYWORDS}}`

**종일 이벤트이면서 키워드가 매칭되면 → 휴일로 판단하고 즉시 종료한다. 아무 메시지도 보내지 않는다.**

반차(오전반차, 오후반차 등)는 종일 이벤트가 아니므로 휴일로 판단하지 않는다. 정상적으로 다음 단계를 진행한다.

---

### 2단계: Linear 이슈 수집

`list_issues`로 이슈를 수집한다.

#### 오늘 할 일 (Todo + In Progress)

다음 조건으로 조회한다:
- assignee: 나 (현재 사용자)
- status: Todo, In Progress
{{LINEAR_SCOPE_FILTER}}

#### 어제 한 일 (Done)

다음 조건으로 조회한다:
- assignee: 나 (현재 사용자)
- status: Done
- completedAt: 어제 범위
{{LINEAR_SCOPE_FILTER}}

**월요일인 경우:** Done 이슈의 completedAt 범위를 지난 금요일 00:00 ~ 오늘 {{REPLY_TIME}} 으로 확장한다. 주말 동안 완료한 이슈도 포함하기 위함이다.

**평일(화~금)인 경우:** Done 이슈의 completedAt 범위는 어제 00:00 ~ 오늘 {{REPLY_TIME}} 이다.

수집 결과가 부족하거나 추가 맥락이 필요하면 `research`로 보충 조회한다.

---

### 3단계: Slack 봇 스레드 찾기

각 채널에 대해 순서대로 처리한다.

{{CHANNELS_CONFIG}}

각 채널마다:

1. `slack_read_channel`로 해당 채널의 최근 메시지를 읽는다.
2. 오늘 날짜에 해당하는 봇의 메시지를 찾는다. 봇 메시지는 보통 {{BOT_TIME}} 경에 게시된다.
3. 봇 메시지의 thread_ts(스레드 타임스탬프)를 확보한다.

**봇이 오늘 스레드를 올리지 않은 경우 → 해당 채널은 스킵한다. 에러를 발생시키지 않는다.**

---

### 4단계: 답글 전송

3단계에서 확보한 각 스레드에 `slack_reply_to_thread`로 답글을 전송한다.

메시지 포맷:

```
{{USER_NAME}}
:clipboard: *오늘 할 일*
• PROJ-123 이슈 제목
• PROJ-456 이슈 제목

:white_check_mark: *어제 한 일*
• PROJ-789 이슈 제목
```

작성 규칙:
- 각 이슈는 `• 이슈식별자 이슈제목` 형태로 한 줄씩 나열한다.
- 이슈식별자는 Linear의 팀 prefix + 번호 (예: `PROJ-123`) 형태를 그대로 사용한다.
- 해당 섹션에 이슈가 0개이면 `• 없음`으로 표시한다.
- 월요일에는 "어제 한 일" 대신 "지난주 한 일"로 라벨을 변경한다.
- Slack mrkdwn 문법을 사용한다 (이모지 shortcode, `*bold*`).
