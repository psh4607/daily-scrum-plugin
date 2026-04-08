---
description: 데일리 스크럼 자동 답글 설정 위자드 — MCP 연결 확인, Slack 봇 탐지, Linear 범위, 휴일 패턴 설정 후 RemoteTrigger 스케줄 생성
---

# Setup 위자드

유저와 대화하며 데일리 스크럼 자동 답글을 설정하는 5단계 결정적 위자드이다.
각 단계는 반드시 완료 + 유저 컨펌 후에만 다음 단계로 진행한다.

## session_context 규칙

RemoteTrigger의 `job_config.ccr.session_context`에는 **`model` 필드만** 포함한다. 그 외 필드(`allowed_tools`, `sources`, `permissions` 등)를 절대로 포함하지 않는다.

- **허용**: `"session_context": {"model": "claude-sonnet-4-6"}`
- **금지**: `allowed_tools`, `sources`, `permissions`, `max_tokens` 등 모든 추가 필드

이 규칙은 create와 update 모두에 적용된다.

## HARD GATE 규칙

- 각 단계 끝에 `--- GATE ---` 표시가 있다.
- 유저가 명시적으로 컨펌해야 다음 단계로 진행한다.
- 컨펌 표현: "다음", "확인", "ㅇㅇ", "좋아", "ㄱㄱ", "넵", "OK" 등 긍정 표현.
- 컨펌 없이 다음 단계로 넘어가지 않는다. 절대로.
- 유저가 수정을 요청하면 해당 단계 내에서 수정 후 다시 컨펌을 받는다.

## 수집할 설정값

위자드를 진행하며 아래 8개 값을 수집한다. 최종적으로 `prompts/schedule-template.md`의 플레이스홀더를 치환하는 데 사용한다.

| 플레이스홀더 | 수집 단계 |
|---|---|
| `{{CHANNEL_NAMES}}` | Step 2 |
| `{{CHANNELS_CONFIG}}` | Step 2 |
| `{{BOT_NAME}}` | Step 2 |
| `{{BOT_TIME}}` | Step 2 |
| `{{REPLY_TIME}}` | Step 2 |
| `{{LINEAR_SCOPE_DESC}}` | Step 3 |
| `{{LINEAR_SCOPE_FILTER}}` | Step 3 |
| `{{HOLIDAY_KEYWORDS}}` | Step 4 |

---

## Step 1: MCP 연결 확인

3개 MCP 커넥터가 정상 연결되었는지 **실제 API 호출**로 확인한다. 단순히 도구 목록을 확인하는 것이 아니라, 실제로 호출하여 응답을 받아야 한다.

### 1-1. 연결 테스트 실행

다음 3개 호출을 수행한다:

1. **Linear**: `claude_ai_Linear`의 `list_teams` 호출
2. **Google Calendar**: `claude_ai_Google_Calendar`의 `gcal_list_calendars` 호출
3. **Slack**: `claude_ai_Slack`의 `slack_search_channels` 호출 (query: `"a"`)

### 1-2. 결과 표 제시

결과를 다음 형식으로 유저에게 보여준다:

```
| 서비스 | 상태 |
|--------|------|
| Linear | ✅ 연결됨 / ❌ 미연결 |
| Google Calendar | ✅ 연결됨 / ❌ 미연결 |
| Slack | ✅ 연결됨 / ❌ 미연결 |
```

### 1-3. 미연결 서비스가 있는 경우

미연결 서비스가 1개라도 있으면:

> 미연결 서비스가 있습니다. 아래 방법으로 연결해주세요:
> - **Claude Desktop**: Settings → Integrations에서 해당 서비스 연결
> - **claude.ai**: Settings → Connected Apps에서 해당 서비스 연결
>
> 연결 완료 후 "완료"라고 알려주세요.

유저가 "완료"라고 하면 **실패한 서비스만** 다시 호출하여 재확인한다. 이미 성공한 서비스는 재확인하지 않는다.

### --- GATE ---

**3개 서비스 모두 ✅** 상태에서 유저가 컨펌해야 Step 2로 진행한다.

---

## Step 2: Slack 채널 / 봇 탐지

### 2-1. 채널 확인

유저에게 질문한다:

> 데일리 스크럼을 올리는 Slack 채널 이름을 알려주세요.

유저가 채널 이름을 알려주면 `claude_ai_Slack`의 `slack_search_channels`로 검색한다.

- 검색 결과에서 매칭되는 채널을 찾아 유저에게 확인한다.
- 매칭되는 채널이 여러 개면 번호로 선택하게 한다.
- 못 찾으면 다시 질문한다.

수집값: 채널 이름 → `{{CHANNEL_NAMES}}`에 사용

### 2-2. 봇 탐지

`claude_ai_Slack`의 `slack_read_channel`로 해당 채널의 최근 7일 메시지를 조회한다.

조회한 메시지에서 **주기적으로 반복되는 봇 메시지 패턴**을 탐지한다:
- 매일 같은 시간대에 게시되는 메시지
- 봇/앱이 보낸 메시지 (사람이 아닌)
- "데일리 스크럼", "standup", "daily" 등 키워드를 포함하는 메시지

탐지 결과에 따라:

- **봇 1개 탐지**: 봇 이름과 게시 시각을 보여주고 바로 컨펌 요청
- **봇 여러 개 탐지**: 번호 목록을 보여주고 선택하게 함
- **못 찾음**: 유저에게 직접 입력 요청

> 봇 이름과 대략적인 게시 시각을 알려주세요. (예: "Standup Bot, 오전 9:30")

수집값: 봇 이름 → `{{BOT_NAME}}`, 봇 게시 시각 → `{{BOT_TIME}}`

### 2-3. 답글 시각 설정

유저에게 답글 시각을 질문한다. 기본값 제안:

> 답글을 올릴 시각을 알려주세요. (기본: 10:00 AM KST)

유저가 시각을 입력하거나 기본값을 수락하면 수집한다.

**시각 안전 가드**: 답글 시각이 봇 게시 시각({{BOT_TIME}}) 이전인 경우 경고한다:

> ⚠️ 답글 시각(HH:MM)이 봇 게시 시각(HH:MM)보다 이릅니다. 봇이 아직 오늘 스레드를 올리지 않은 상태에서 실행되면 전날 스레드에 답글이 올라갈 수 있습니다.
>
> 그래도 이 시각으로 설정하시겠습니까?

수집값: 답글 시각 → `{{REPLY_TIME}}`

### 2-4. Step 2 요약

수집한 내용을 요약하여 유저에게 보여준다:

```
📋 Slack 설정 요약
- 채널: #채널이름
- 봇: 봇이름 (게시 시각: HH:MM KST)
- 답글 시각: HH:MM KST
```

### --- GATE ---

**채널 + 봇 + 답글 시각** 3개 모두 수집 완료 상태에서 유저가 컨펌해야 Step 3으로 진행한다.

---

## Step 3: Linear 범위 설정

유저에게 Linear 이슈 조회 범위를 선택하게 한다. 선택지를 제시한다:

> Linear 이슈를 어떤 범위로 조회할까요?
>
> **A) 나에게 할당된 이슈 전체** (추천)
> B) 특정 팀의 이슈만
> C) 특정 프로젝트의 이슈만
> D) 현재 사이클의 이슈만
>
> 기본값: A

### 선택별 처리

**A) 나에게 할당된 이슈 전체**
- 추가 질문 없음.
- `{{LINEAR_SCOPE_DESC}}`: `나에게 할당된 이슈 전체`
- `{{LINEAR_SCOPE_FILTER}}`: (빈 문자열 — 필터 없음, assignee만 적용)

**B) 특정 팀**
- `claude_ai_Linear`의 `list_teams`를 호출하여 팀 목록을 보여준다.
- 유저가 팀을 선택하면 해당 팀 이름과 ID를 기록한다.
- `{{LINEAR_SCOPE_DESC}}`: `팀: {팀이름}`
- `{{LINEAR_SCOPE_FILTER}}`: `- team: {팀이름} (ID: {팀ID})`

**C) 특정 프로젝트**
- `claude_ai_Linear`의 `list_projects`를 호출하여 프로젝트 목록을 보여준다.
- 유저가 프로젝트를 선택하면 해당 프로젝트 이름과 ID를 기록한다.
- `{{LINEAR_SCOPE_DESC}}`: `프로젝트: {프로젝트이름}`
- `{{LINEAR_SCOPE_FILTER}}`: `- project: {프로젝트이름} (ID: {프로젝트ID})`

**D) 현재 사이클**
- `claude_ai_Linear`의 `list_cycles`를 호출하여 활성 사이클 목록을 보여준다.
- 유저가 사이클을 선택하면 해당 사이클 정보를 기록한다.
- `{{LINEAR_SCOPE_DESC}}`: `사이클: {사이클이름}`
- `{{LINEAR_SCOPE_FILTER}}`: `- cycle: {사이클이름} (ID: {사이클ID})`

### 상태 매핑

Linear 표준 상태 카테고리(Todo / In Progress / Done)를 자동 적용한다. 별도 컨펌 없이 다음과 같이 안내한다:

> 상태 매핑은 Linear 표준 카테고리를 따릅니다:
> - **오늘 할 일**: Todo, In Progress 상태의 이슈
> - **어제 한 일**: Done 상태의 이슈 (완료일 기준)

### --- GATE ---

**Linear 범위 선택** 완료 상태에서 유저가 컨펌해야 Step 4로 진행한다.

---

## Step 4: 캘린더 휴일 패턴

### 4-1. 공휴일 캘린더 확인

`claude_ai_Google_Calendar`의 `gcal_list_calendars`를 호출하여 캘린더 목록에서 "대한민국의 휴일" (또는 "Holidays in South Korea")이 있는지 확인한다.

- **있으면**: 유저에게 알린다.
  > 공휴일 캘린더("대한민국의 휴일")를 발견했습니다. 공휴일에는 자동으로 스크럼을 스킵합니다.
- **없으면**: 유저에게 알린다.
  > 공휴일 캘린더를 찾지 못했습니다. 공휴일 판단은 Claude가 날짜를 보고 직접 판단합니다.

### 4-2. 개인 휴가 패턴 탐지

`claude_ai_Google_Calendar`의 `gcal_list_events`로 최근 3개월 이벤트를 조회한다.
- 기간: 오늘로부터 3개월 전 ~ 오늘
- 종일 이벤트(all-day event)만 필터링

종일 이벤트 중에서 **휴가/부재를 나타내는 패턴**을 탐지한다:
- 키워드: 휴가, 연차, PTO, flex, vacation, day off, 바쁨
- 반복되는 패턴이 있으면 해당 키워드를 추출한다

탐지 결과에 따라:

**패턴을 찾은 경우**:
> 캘린더에서 다음 휴가 패턴을 발견했습니다:
> - 키워드: "연차", "PTO" (예: 3/15 "연차", 2/28 "PTO")
>
> 이 키워드가 포함된 종일 이벤트가 있는 날에는 스크럼을 스킵합니다.
> 키워드를 수정하거나 추가하시겠습니까?

**패턴을 못 찾은 경우**:
> 최근 3개월 캘린더에서 휴가 패턴을 찾지 못했습니다.
>
> A) 공휴일만 스킵 (추가 키워드 없음)
> B) 휴가 키워드를 직접 입력
>
> 직접 입력하실 키워드가 있으면 알려주세요. (예: "연차, PTO, 휴가")

### 참고 사항

반차(오전반차, 오후반차)는 종일 이벤트가 아니므로 휴일로 판단하지 않는다. 이 점을 유저에게 안내한다:

> 참고: 반차(오전/오후)는 종일 이벤트가 아니므로 정상적으로 스크럼이 게시됩니다.

### 수집값 정리

공휴일 캘린더 존재 여부 + 유저가 확정한 키워드 목록을 합쳐서 `{{HOLIDAY_KEYWORDS}}`를 구성한다.

- 공휴일 캘린더 있음: 키워드에 별도 추가 불필요 (schedule-template.md의 1단계에서 "대한민국의 휴일" 캘린더를 직접 조회)
- 개인 휴가 키워드: 유저가 확정한 키워드를 쉼표 구분으로 나열

예: `휴가, 연차, PTO, vacation`

### --- GATE ---

**휴일 기준**(공휴일 캘린더 + 개인 휴가 키워드) 확정 상태에서 유저가 컨펌해야 Step 5로 진행한다.

---

## Step 5: 스케줄 생성

### 5-1. 최종 설정 요약

지금까지 수집한 모든 설정을 요약하여 유저에게 보여준다:

```
🔧 최종 설정 요약

[Slack]
- 채널: #채널이름
- 봇: 봇이름 (게시 시각: HH:MM KST)
- 답글 시각: HH:MM KST

[Linear]
- 범위: (선택한 범위 설명)
- 상태 매핑: Todo/In Progress → 오늘 할 일, Done → 어제 한 일

[캘린더]
- 공휴일: 대한민국의 휴일 캘린더 사용 / Claude 직접 판단
- 휴가 키워드: 키워드1, 키워드2, ...

[스케줄]
- 실행 시각: HH:MM KST (HH:MM UTC)
- 실행 주기: 평일(월~금)
```

### --- GATE ---

유저가 최종 설정 요약을 확정해야 스케줄 생성을 실행한다. 컨펌 없이 절대 진행하지 않는다.

### 5-2. 스케줄 생성 실행

유저가 컨펌하면 다음 순서로 스케줄을 생성한다:

**1) 템플릿 읽기**

Read 도구로 `prompts/schedule-template.md`를 읽는다. 경로는 이 파일(commands/setup.md) 기준 `../prompts/schedule-template.md`이다.

**2) 플레이스홀더 치환**

읽어온 템플릿에서 8개 플레이스홀더를 수집한 설정값으로 치환한다:

| 플레이스홀더 | 치환값 |
|---|---|
| `{{CHANNEL_NAMES}}` | 채널 이름 (예: `#daily-standup`) |
| `{{CHANNELS_CONFIG}}` | 채널별 봇 설정 블록 (아래 형식) |
| `{{BOT_NAME}}` | 탐지/입력된 봇 이름 |
| `{{BOT_TIME}}` | 봇 게시 시각 (예: `09:30 AM KST`) |
| `{{REPLY_TIME}}` | 답글 시각 (예: `10:00 AM KST`) |
| `{{LINEAR_SCOPE_DESC}}` | Linear 범위 설명 |
| `{{LINEAR_SCOPE_FILTER}}` | Linear 범위 필터 조건 (또는 빈 문자열) |
| `{{HOLIDAY_KEYWORDS}}` | 쉼표 구분 키워드 목록 |

`{{CHANNELS_CONFIG}}` 형식:

```
- 채널: #채널이름
  봇: 봇이름
  봇 게시 시각: HH:MM KST
```

**3) KST → UTC 변환**

답글 시각을 KST에서 UTC로 변환한다. KST = UTC+9.

예시:
- 10:00 KST → 01:00 UTC → cron: `0 1 * * 1-5`
- 09:30 KST → 00:30 UTC → cron: `30 0 * * 1-5`
- 08:00 KST → 23:00 UTC (전날) → cron: `0 23 * * 0-4` (일~목 = KST 기준 월~금)

주의: KST 시각을 UTC로 변환할 때 날짜가 바뀔 수 있다. UTC 기준 요일도 그에 맞게 조정한다.
- KST 기준 월~금이 UTC로 전날이 되는 경우: cron 요일을 `0-4` (일~목)로 설정한다.
- KST 기준 월~금이 UTC로 같은 날인 경우: cron 요일을 `1-5` (월~금)로 설정한다.

**4) 기존 트리거 조회 (MCP 커넥터 + 환경 탐색)**

RemoteTrigger에 MCP 커넥터 정보와 environment_id가 필요하다. 다음 순서로 수집한다:

1. `RemoteTrigger action:"list"`로 기존 트리거 목록을 조회한다.
2. **기존 트리거가 있으면:**
   - `mcp_connections`에서 Linear, Google-Calendar, Slack 커넥터 정보(connector_uuid, name, url)를 추출한다.
   - `job_config.ccr.environment_id`를 확인한다. git source가 없는 환경(기본 클라우드 환경)의 ID를 우선 사용한다.
   - 기본 클라우드 환경 판별: `session_context`에 `sources` 필드가 없거나 빈 배열인 트리거의 environment_id를 선택한다.
   - 모든 트리거가 git source를 포함한 환경이면 → 아래 "환경 생성 안내"를 진행한다.
3. **기존 트리거가 없거나 적합한 환경이 없으면** → 유저에게 안내한다:

   > "기본 클라우드 환경"이 필요합니다. Claude Desktop에서 설정해주세요:
   > 1. Claude Desktop → 왼쪽 사이드바 → **Scheduled** 메뉴
   > 2. 아무 스케줄의 **환경**(Environment)을 "기본 클라우드 환경"으로 변경
   >    (또는 새 스케줄을 하나 만들며 "기본 클라우드 환경" 선택)
   > 3. "완료"라고 알려주세요.

   유저가 "완료"라고 하면 다시 `RemoteTrigger action:"list"`로 조회하여 환경 ID와 커넥터 정보를 추출한다.

**5) RemoteTrigger로 스케줄 등록**

`RemoteTrigger action:"create"`를 호출하여 스케줄을 등록한다. body 형식:

```json
{
  "name": "daily-scrum-{채널이름}",
  "cron_expression": "{위에서 계산한 UTC 기반 cron}",
  "enabled": true,
  "persist_session": false,
  "mcp_connections": [
    {"connector_uuid": "{추출한 UUID}", "name": "Google-Calendar", "permitted_tools": [], "url": "https://gcal.mcp.claude.com/mcp"},
    {"connector_uuid": "{추출한 UUID}", "name": "Linear", "permitted_tools": [], "url": "https://mcp.linear.app/mcp"},
    {"connector_uuid": "{추출한 UUID}", "name": "Slack", "permitted_tools": [], "url": "https://mcp.slack.com/mcp"}
  ],
  "job_config": {
    "ccr": {
      "environment_id": "{탐색한 기본 클라우드 환경 ID}",
      "events": [
        {
          "data": {
            "message": {"content": "{치환된 schedule-template.md 전체 내용}", "role": "user"},
            "type": "user",
            "uuid": "{임의 UUID 생성}"
          }
        }
      ],
      "session_context": {
        "model": "claude-sonnet-4-6"
      }
    }
  }
}
```

> **주의**: `session_context`에는 반드시 `model` 필드만 포함한다. `allowed_tools`, `sources` 등 다른 필드를 추가하면 스케줄 실행 시 MCP 도구 접근이 제한되어 실패한다. 위 `## session_context 규칙` 참조.

**environment_id 만료 대응:**

`RemoteTrigger action:"create"` 호출 결과에 따라 분기한다:

- **성공 (HTTP 200)**: environment_id가 유효하다. 생성된 trigger_id를 기록하고 다음 단계로 진행한다.
- **실패 (에러 응답)**: environment_id가 만료되었을 가능성이 높다. 아래 환경 복구 플로우를 실행한다.

**주의**: 기존 트리거에 `action:"run"`을 실행하지 않는다. 다른 트리거를 run하면 GitHub 이슈 생성, Slack 메시지 전송 등 되돌릴 수 없는 부작용이 발생한다.

**환경 복구 플로우:**

> 스케줄 생성에 실패했습니다. 환경이 만료된 것으로 보입니다. 새 환경을 설정해주세요:
> 1. Claude Desktop → 왼쪽 사이드바 → **예약된 작업**(Scheduled) 메뉴
> 2. 아무 스케줄의 **환경**(Environment)을 "기본 클라우드 환경"으로 변경
>    (또는 새 스케줄을 하나 만들며 "기본 클라우드 환경" 선택)
> 3. "완료"라고 알려주세요.

유저가 "완료"라고 하면:
1. `RemoteTrigger action:"list"`로 다시 조회하여 새 environment_id를 추출한다.
2. 새 environment_id로 `RemoteTrigger action:"create"`를 다시 시도한다.
3. 성공 시 다음 단계로 진행한다. 실패 시 위 안내를 반복한다 (최대 2회).

**6) 생성 후 session_context 검증**

스케줄 생성 직후 `RemoteTrigger action:"get"` trigger_id:{생성된 trigger_id}로 조회한다.

응답의 `job_config.ccr.session_context`를 확인한다:
- `model` 외 다른 필드(`allowed_tools`, `sources` 등)가 포함되어 있으면 → 즉시 교정한다.
- 교정: `RemoteTrigger action:"update"` trigger_id:{trigger_id}로 전체 job_config를 다시 전송한다. 기존 트리거의 `environment_id`, `events`, `mcp_connections` 값을 유지하되 `session_context`는 `{"model": "claude-sonnet-4-6"}`만 포함한다.
- 교정 후 다시 get으로 확인한다. 여전히 오염되어 있으면 1회 더 교정을 시도한다 (최대 2회).
- `session_context`에 `model`만 있으면 → 교정 불필요, 다음 단계로 진행.

### 5-3. 완료 안내

스케줄 등록이 완료되면 응답에서 `trigger_id`를 확인하고 유저에게 안내한다:

> 스케줄이 생성되었습니다!
>
> - **트리거 ID**: {trigger_id}
> - **스케줄 이름**: daily-scrum-{채널이름}
> - **실행 시각**: HH:MM KST (매주 월~금)
> - **다음 실행**: (가장 가까운 다음 평일 시각)
>
> 스케줄 확인: claude.ai → Settings → Scheduled에서 등록된 스케줄을 확인할 수 있습니다.
> 설정 수정: `/daily-scrum:modify` 커맨드로 채널, 시간, Linear 범위 등을 변경할 수 있습니다.

### 5-4. 선택적 테스트 실행

유저에게 테스트 실행 여부를 물어본다:

> 스케줄이 정상 동작하는지 바로 테스트해볼 수 있습니다.
> 테스트하면 **실제로 Slack에 답글이 올라갑니다**. 테스트할까요?

유저가 동의하면:
1. `RemoteTrigger action:"run"` trigger_id:{방금 생성한 trigger_id}를 호출한다.
2. 성공 시: "테스트 실행이 시작되었습니다. 잠시 후 Slack을 확인해보세요."
3. 실패 시: environment_id 만료 가능성을 안내하고 위 환경 복구 플로우를 실행한다.

유저가 거부하면: "다음 스케줄 실행(내일 {REPLY_TIME})에 정상 동작을 확인해주세요."로 안내한다.
