---
description: 데일리 스크럼 스케줄 수정 — 채널 변경/추가, 시간 변경, Linear 범위 변경, 휴일 기준 변경, 일시 중지/삭제
---

# Modify 스케줄

유저와 대화하며 기존 데일리 스크럼 스케줄을 수정하는 플로우이다.

## session_context 규칙

RemoteTrigger의 `job_config.ccr.session_context`에는 **`model` 필드만** 포함한다. 그 외 필드(`allowed_tools`, `sources`, `permissions` 등)를 절대로 포함하지 않는다.

- **허용**: `"session_context": {"model": "claude-sonnet-4-6"}`
- **금지**: `allowed_tools`, `sources`, `permissions`, `max_tokens` 등 모든 추가 필드

이 규칙은 update 호출 시에도 동일하게 적용된다.

## 알려진 제한사항

- 수정 메뉴 F(표시 이름 변경)와 G(스케줄 일시 중지/재개)의 본문 섹션이 메뉴 라벨과 일치하지 않는 문제가 있다. 이 불일치는 별도 이슈로 수정한다.

---

## Step 1: 기존 스케줄 조회

### 1-1. 스케줄 목록 조회

`RemoteTrigger action:"list"`로 등록된 트리거 목록을 조회한다.

이름에 `daily-scrum`이 포함된 트리거를 필터링한다. `enabled: false`인 트리거는 "일시 중지됨"으로 표시한다.

### 1-2. 결과 분기

**스케줄을 찾은 경우:**

트리거의 `job_config.ccr.events[0].data.message.content`에서 프롬프트를 추출하고, 그 안의 `## 설정 정보` 섹션을 파싱하여 현재 설정값을 추출한다. 트리거의 `id`(trigger_id)와 `cron_expression`도 기록해둔다:

- 채널 목록
- 봇 이름 / 봇 게시 시각
- 답글 시각
- Linear 범위
- 휴가 판단 키워드

추출한 설정을 유저에게 요약한다:

```
📋 현재 설정
- 트리거 ID: {trigger_id}
- 스케줄 이름: {스케줄 이름}
- 상태: 활성 / 일시 중지됨
- 채널: {채널 목록}
- 봇: {봇 이름} (게시 시각: {봇 시각})
- 답글 시각: {답글 시각}

- Linear 범위: {범위 설명}
- 휴가 키워드: {키워드 목록}
```

이어서 수정 메뉴를 제시한다 (Step 2로).

**스케줄을 찾지 못한 경우:**

> 등록된 데일리 스크럼 스케줄이 없습니다. `/daily-scrum:setup`으로 먼저 설정해주세요.

여기서 종료한다.

---

## Step 2: 수정 항목 선택

유저에게 수정 메뉴를 제시한다:

> 어떤 항목을 수정하시겠습니까?
>
> A) 채널 변경
> B) 채널 추가 (다중 채널)
> C) 답글 시각 변경
> D) Linear 범위 변경
> E) 휴일 기준 변경
> F) 표시 이름 변경
> G) 스케줄 일시 중지 / 재개
> H) 스케줄 삭제

유저가 선택하면 해당 항목의 플로우를 실행한다.

---

### A) 채널 변경

기존 채널을 다른 채널로 교체한다.

1. 유저에게 새 채널명을 입력받는다.

   > 변경할 Slack 채널 이름을 알려주세요.

2. `claude_ai_Slack`의 `slack_search_channels`로 검색한다.
   - 매칭되는 채널이 여러 개면 번호로 선택하게 한다.
   - 못 찾으면 다시 질문한다.

3. `claude_ai_Slack`의 `slack_read_channel`로 해당 채널의 최근 7일 메시지를 조회하여 봇을 재탐지한다.
   - 매일 같은 시간대에 게시되는 봇 메시지 패턴을 찾는다.
   - 봇 1개 탐지 → 봇 이름과 게시 시각을 보여주고 컨펌.
   - 봇 여러 개 탐지 → 번호 목록에서 선택.
   - 못 찾음 → 유저에게 직접 입력 요청.

4. 변경 내용을 유저에게 보여주고 컨펌을 받는다.

   ```
   📝 채널 변경
   - 기존: #기존채널 (봇: 기존봇, 게시: HH:MM)
   - 변경: #새채널 (봇: 새봇, 게시: HH:MM)
   ```

5. **시각 안전 가드**: 봇 게시 시각이 변경된 경우, 기존 답글 시각이 새 봇 게시 시각보다 이전인지 확인한다.

   이전이면 경고한다:
   > ⚠️ 답글 시각({답글 시각})이 새 봇 게시 시각({봇 게시 시각})보다 이릅니다. 봇이 아직 오늘 스레드를 올리지 않은 상태에서 실행되면 전날 스레드에 답글이 올라갈 수 있습니다.
   >
   > 그래도 이대로 진행하시겠습니까?

6. 컨펌 후 → **스케줄 업데이트** (아래 "스케줄 업데이트 패턴" 참조).

---

### B) 채널 추가 (다중 채널)

기존 채널은 유지하면서 새 채널을 추가한다.

1. 유저에게 추가할 채널명을 입력받는다.

   > 추가할 Slack 채널 이름을 알려주세요.

2. `claude_ai_Slack`의 `slack_search_channels`로 검색하여 채널을 확인한다.

3. `claude_ai_Slack`의 `slack_read_channel`로 해당 채널의 최근 7일 메시지를 조회하여 봇을 탐지한다.
   - 봇 1개 탐지 → 봇 이름과 게시 시각을 보여주고 컨펌.
   - 봇 여러 개 탐지 → 번호 목록에서 선택.
   - 못 찾음 → 유저에게 직접 입력 요청.

4. 변경 내용을 유저에게 보여주고 컨펌을 받는다.

   ```
   📝 채널 추가
   - 기존 채널: #기존채널1
   - 추가 채널: #새채널 (봇: 봇이름, 게시: HH:MM)
   ```

5. 컨펌 후 → 스케줄 프롬프트의 `CHANNELS_CONFIG`에 새 채널 정보를 추가하고, `CHANNEL_NAMES`에 채널 이름을 추가하여 **스케줄 업데이트** (아래 "스케줄 업데이트 패턴" 참조).

---

### C) 답글 시각 변경

1. 유저에게 새 답글 시각을 입력받는다.

   > 새 답글 시각을 알려주세요. (예: 10:00 AM KST, 09:30)

2. **시각 안전 가드**: 새 답글 시각이 현재 봇 게시 시각보다 이전인지 확인한다.

   이전이면 경고한다:
   > ⚠️ 답글 시각({새 답글 시각})이 봇 게시 시각({봇 게시 시각})보다 이릅니다. 봇이 아직 오늘 스레드를 올리지 않은 상태에서 실행되면 전날 스레드에 답글이 올라갈 수 있습니다.
   >
   > 그래도 이 시각으로 설정하시겠습니까?

3. **KST → UTC 변환**: 새 답글 시각을 UTC로 변환한다. KST = UTC+9.

   예시:
   - 10:00 KST → 01:00 UTC → cron: `0 1 * * 1-5`
   - 09:30 KST → 00:30 UTC → cron: `30 0 * * 1-5`
   - 08:00 KST → 23:00 UTC (전날) → cron: `0 23 * * 0-4` (일~목 = KST 기준 월~금)

   주의: KST 시각을 UTC로 변환할 때 날짜가 바뀔 수 있다. UTC 기준 요일도 그에 맞게 조정한다.
   - KST 기준 월~금이 UTC로 전날이 되는 경우: cron 요일을 `0-4` (일~목)로 설정한다.
   - KST 기준 월~금이 UTC로 같은 날인 경우: cron 요일을 `1-5` (월~금)로 설정한다.

4. 변경 내용을 유저에게 보여주고 컨펌을 받는다.

   ```
   📝 답글 시각 변경
   - 기존: HH:MM KST
   - 변경: HH:MM KST (HH:MM UTC)
   ```

5. 컨펌 후 → `RemoteTrigger action:"update"`로 cron_expression과 프롬프트(REPLY_TIME 갱신)를 변경한다 (아래 "스케줄 업데이트 패턴" 참조).

---

### D) Linear 범위 변경

setup Step 3과 동일한 선택지를 제시한다.

1. 유저에게 새 Linear 범위를 선택하게 한다:

   > Linear 이슈를 어떤 범위로 조회할까요?
   >
   > **A) 나에게 할당된 이슈 전체** (추천)
   > B) 특정 팀의 이슈만
   > C) 특정 프로젝트의 이슈만
   > D) 현재 사이클의 이슈만

2. 선택별 처리:

   **A) 나에게 할당된 이슈 전체**
   - 추가 질문 없음.
   - `LINEAR_SCOPE_DESC`: `나에게 할당된 이슈 전체`
   - `LINEAR_SCOPE_FILTER`: (빈 문자열)

   **B) 특정 팀**
   - `claude_ai_Linear`의 `list_teams`를 호출하여 팀 목록을 보여준다.
   - 유저가 팀을 선택하면 해당 팀 이름과 ID를 기록한다.
   - `LINEAR_SCOPE_DESC`: `팀: {팀이름}`
   - `LINEAR_SCOPE_FILTER`: `- team: {팀이름} (ID: {팀ID})`

   **C) 특정 프로젝트**
   - `claude_ai_Linear`의 `list_projects`를 호출하여 프로젝트 목록을 보여준다.
   - 유저가 프로젝트를 선택하면 해당 프로젝트 이름과 ID를 기록한다.
   - `LINEAR_SCOPE_DESC`: `프로젝트: {프로젝트이름}`
   - `LINEAR_SCOPE_FILTER`: `- project: {프로젝트이름} (ID: {프로젝트ID})`

   **D) 현재 사이클**
   - `claude_ai_Linear`의 `list_cycles`를 호출하여 활성 사이클 목록을 보여준다.
   - 유저가 사이클을 선택하면 해당 사이클 정보를 기록한다.
   - `LINEAR_SCOPE_DESC`: `사이클: {사이클이름}`
   - `LINEAR_SCOPE_FILTER`: `- cycle: {사이클이름} (ID: {사이클ID})`

3. 변경 내용을 유저에게 보여주고 컨펌을 받는다.

   ```
   📝 Linear 범위 변경
   - 기존: {기존 범위}
   - 변경: {새 범위}
   ```

4. 컨펌 후 → 프롬프트의 `LINEAR_SCOPE_DESC`와 `LINEAR_SCOPE_FILTER`를 갱신하여 **스케줄 업데이트** (아래 "스케줄 업데이트 패턴" 참조).

---

### E) 휴일 기준 변경

setup Step 4와 동일한 캘린더 패턴 탐지를 수행한다.

1. `claude_ai_Google_Calendar`의 `gcal_list_calendars`를 호출하여 "대한민국의 휴일" 캘린더 존재 여부를 확인한다.

   - **있으면**: > 공휴일 캘린더("대한민국의 휴일")를 발견했습니다. 공휴일에는 자동으로 스크럼을 스킵합니다.
   - **없으면**: > 공휴일 캘린더를 찾지 못했습니다. 공휴일 판단은 Claude가 날짜를 보고 직접 판단합니다.

2. `claude_ai_Google_Calendar`의 `gcal_list_events`로 최근 3개월 이벤트를 조회한다.
   - 기간: 오늘로부터 3개월 전 ~ 오늘
   - 종일 이벤트만 필터링
   - 휴가/부재 키워드 탐지: 휴가, 연차, PTO, flex, vacation, day off, 바쁨

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

   반차 안내도 표시한다:
   > 참고: 반차(오전/오후)는 종일 이벤트가 아니므로 정상적으로 스크럼이 게시됩니다.

3. 변경 내용을 유저에게 보여주고 컨펌을 받는다.

   ```
   📝 휴일 기준 변경
   - 기존 키워드: {기존 키워드}
   - 변경 키워드: {새 키워드}
   ```

4. 컨펌 후 → 프롬프트의 `HOLIDAY_KEYWORDS`를 갱신하여 **스케줄 업데이트** (아래 "스케줄 업데이트 패턴" 참조).

---

### F) 스케줄 일시 중지 / 재개

트리거의 `enabled` 상태에 따라 동작이 달라진다.

**일시 중지 (enabled: true → false):**

1. 유저에게 확인한다.

   > 데일리 스크럼 스케줄을 일시 중지하시겠습니까? 프롬프트 설정은 보존됩니다.

2. 컨펌 후 → `RemoteTrigger action:"update"` trigger_id:{trigger_id} body:{enabled: false}로 비활성화한다.

3. 유저에게 안내한다:

   > ⏸️ 스케줄이 일시 중지되었습니다. 프롬프트와 설정은 그대로 보존되어 있습니다.
   >
   > 재개하려면 `/daily-scrum:modify`를 다시 실행하세요.

**재개 (enabled: false → true):**

1. Step 1에서 일시 중지된 트리거(enabled: false)를 발견하면 유저에게 안내한다:

   > 일시 중지된 스케줄을 발견했습니다. 재개하시겠습니까?

2. 컨펌 후 → `RemoteTrigger action:"update"` trigger_id:{trigger_id} body:{enabled: true}로 활성화한다.

3. 유저에게 안내한다:

   > ▶️ 스케줄이 재개되었습니다.
   > - **스케줄 이름**: {스케줄 이름}
   > - **실행 시각**: HH:MM KST (매주 월~금)

---

### H) 스케줄 삭제

RemoteTrigger API에는 삭제 기능이 없으므로, 비활성화 + 안내로 대체한다.

1. 유저에게 확인한다.

   > ⚠️ RemoteTrigger는 삭제 API를 제공하지 않아, 비활성화로 처리됩니다.
   > 완전히 삭제하려면 claude.ai → Settings → Scheduled에서 직접 삭제해주세요.
   >
   > 비활성화를 진행하시겠습니까?

2. 유저가 명시적으로 확인하면 `RemoteTrigger action:"update"` trigger_id:{trigger_id} body:{enabled: false}로 비활성화한다.

3. 완료 안내:

   > 🗑️ 스케줄 "{스케줄 이름}"이 비활성화되었습니다.
   > 완전 삭제가 필요하면 claude.ai → Settings → Scheduled에서 해당 트리거를 삭제해주세요.
   > 다시 사용하려면 `/daily-scrum:modify`에서 재개하거나, `/daily-scrum:setup`으로 새로 설정할 수 있습니다.

---

## Step 3: 완료 안내

수정 항목 A~F 완료 후 다음을 표시한다 (G, H는 자체 완료 메시지가 있으므로 제외):

> ✅ 설정이 업데이트되었습니다.

변경사항 요약:

```
📝 변경사항
- {변경 항목}: {기존값} → {새값}
```

업데이트된 설정 전체 요약:

```
📋 업데이트된 설정
- 트리거 ID: {trigger_id}
- 스케줄 이름: {스케줄 이름}
- 채널: {채널 목록}
- 봇: {봇 이름} (게시 시각: {봇 시각})
- 답글 시각: {답글 시각}

- Linear 범위: {범위 설명}
- 휴가 키워드: {키워드 목록}
- 실행 주기: 평일(월~금), HH:MM KST
```

> 스케줄 확인: claude.ai → Settings → Scheduled에서 등록된 스케줄을 확인할 수 있습니다.

---

## 스케줄 업데이트 패턴

수정 항목 A~E에서 설정이 변경되면 아래 절차로 기존 트리거를 수정한다.

### 공통 절차: 프롬프트 재생성 + 전체 job_config update

1. Read 도구로 `prompts/schedule-template.md`를 읽는다. 경로는 이 파일(commands/modify.md) 기준 `../prompts/schedule-template.md`이다.
2. 변경된 설정값으로 플레이스홀더를 치환한다. 변경되지 않은 값은 기존 트리거에서 파싱한 값을 그대로 사용한다.
3. `RemoteTrigger action:"update"` trigger_id:{trigger_id}로 트리거를 수정한다.

update body는 **항상 `job_config` 전체를 포함**한다. partial update의 merge 동작(deep merge vs shallow replace)이 불확실하므로, `job_config.ccr`의 모든 필드를 빠짐없이 전송한다:

```json
{
  "job_config": {
    "ccr": {
      "environment_id": "{기존 트리거의 environment_id}",
      "events": [
        {
          "data": {
            "message": {"content": "{치환된 프롬프트 전체}", "role": "user"},
            "type": "user",
            "uuid": "{기존 트리거의 UUID 유지}"
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

- **name**: 채널이 변경된 경우에만 body에 `"name": "daily-scrum-{새채널이름}"` 추가. 그 외에는 생략.
- **session_context**: 반드시 `model` 필드만 포함. `allowed_tools` 등 다른 필드 금지 (위 `## session_context 규칙` 참조).
- **environment_id, events**: 기존 트리거의 값을 재사용. events의 message.content만 새 프롬프트로 교체.

### cron 시각 변경 시 (항목 C)

위 공통 절차를 따르되, body에 `cron_expression`을 추가한다:

```json
{
  "cron_expression": "{새로 계산한 UTC 기반 cron}",
  "job_config": { ... }
}
```

새 답글 시각의 KST → UTC 변환 결과로 cron 표현식을 새로 생성한다.

### update 후 session_context 사후 검증

update 완료 직후 `RemoteTrigger action:"get"` trigger_id:{trigger_id}로 조회한다.

응답의 `job_config.ccr.session_context`를 확인한다:
- `model` 외 다른 필드(`allowed_tools`, `sources` 등)가 포함되어 있으면 → 위 공통 절차의 update를 한 번 더 실행하여 교정한다 (최대 2회).
- `model`만 있으면 → 교정 불필요, Step 3 완료 안내로 진행.
