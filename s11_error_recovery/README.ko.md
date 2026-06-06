# s11: Error Recovery — 에러는 끝이 아니라 재시도의 시작이다

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s09 → s10 → `s11` → [s12](../s12_task_system/) → s13 → ... → s20
> *"에러는 끝이 아니라 재시도의 시작이다"* — 토큰을 늘리고, context를 compact하고, 모델을 전환한다.
>
> **Harness 레이어**: 회복 탄력성(Resilience) — 메인 loop가 에러를 만났을 때 이를 분류하고 복구한다.

---

## 문제 상황

Agent가 잘 돌아가다가 에러로 멈춰버린다:

```
Error: 529 overloaded
```

Agent가 crash한다. 재시도도, 모델 전환도, context 축소도 하지 않고 — 그냥 crash할 뿐이다.

프로덕션에서 API 에러는 일상이다. 가장 흔한 세 가지 실패 양상은 다음과 같다: **output 잘림**(모델이 문장 중간에 토큰이 떨어지는 경우), **context overflow**(compact를 거쳐도 여전히 너무 긴 경우), 그리고 **일시적 실패**(429 rate limiting / 529 overload). 에러를 처리하지 않는 Agent는 살짝만 건드려도 시동이 꺼지는 자동차와 같다.

---

## 해결책

![Error Recovery 개요](images/error-recovery-overview.en.svg)

s10의 loop와 prompt 조립 로직은 그대로 보존된다. 유일한 변경점은 LLM 호출을 try/except로 감싸고, 에러 유형에 따라 서로 다른 복구 경로를 타도록 한 것이다. 복구가 끝나면 `continue`로 다시 맨 위로 돌아가 LLM을 다시 호출한다.

가장 흔한 세 가지 복구 패턴은 다음과 같다(교육용 버전은 429/529만 다루지만, 실제 시스템은 connection 에러, timeout, 클라우드 벤더 credential 캐시 등도 처리한다. CC는 실제로 13개 이상의 reason code를 가지고 있으며, 나머지는 Deep Dive를 참고하라):

| 패턴 | 트리거 | 복구 동작 |
|----------|---------|-----------------|
| Output 잘림 | `max_tokens` | 8K→64K 증액 / continuation prompt |
| Context overflow | `prompt_too_long` | Reactive compact → 재시도 |
| 일시적 실패 | 429 / 529 | Exponential backoff + jitter, 연속 529 시 fallback 모델 전환 |

---

## 동작 방식

### 경로 1: Output 잘림

모델이 문장 중간에 토큰이 떨어지는 경우 — `max_tokens`가 소진된 것이다. 기본값인 8000 토큰으로는 완전한 응답을 담기에 부족하다.

처음 발생하면 `max_tokens`를 8K에서 64K로 증액(8배 공간)하고 동일한 요청을 재시도한다 — 잘린 output은 messages에 추가하지 *않아서* 원래 요청을 그대로 유지한다. 64K로도 여전히 부족하면 잘린 output을 저장하고, 모델에게 멈춘 지점부터 이어서 작성하라는 continuation prompt를 주입한다. 최대 3회까지 시도한다:

```python
if response.stop_reason == "max_tokens":
    # First escalation: don't append truncated output, retry same request
    if not state.has_escalated:
        max_tokens = ESCALATED_MAX_TOKENS
        state.has_escalated = True
        continue  # messages unchanged, same request with more tokens
    # 64K still truncated: save output + continuation prompt
    messages.append({"role": "assistant", "content": response.content})
    if state.recovery_count < MAX_RECOVERY_RETRIES:
        messages.append({"role": "user", "content":
            "Output token limit hit. Resume directly — "
            "no apology, no recap. Pick up mid-thought."})
        state.recovery_count += 1
        continue
    return  # still truncated after 3 continuations
# Normal: append after max_tokens check
messages.append({"role": "assistant", "content": response.content})
```

증액(escalation)은 한 번만 기회를 주고, continuation은 최대 3회까지 허용한다. 그 이후에는 종료한다 — 더 이어 붙여도 의미 있는 output이 나오지 않기 때문이다.

### 경로 2: Context Overflow

LLM이 "context가 너무 길다"(`prompt_too_long`)고 알려온다. s08의 네 가지 compaction 레이어가 이미 모두 실행됐는데도 여전히 한계를 초과한 상황이다.

Reactive compact를 트리거한다 — auto compact보다 더 공격적이다. 교육용 버전은 compaction을 흉내내기 위해 마지막 5개 메시지만 남기지만, 실제 CC는 LLM으로 compact 요약을 생성한 뒤 compact된 메시지 목록으로 재시도한다. Compact 후 재시도한다. 하지만 한 번 compact한 뒤에도 여전히 한계를 초과한다면, 남은 선택지는 종료뿐이다 — 다시 compact해도 더 작아지지 않기 때문이다:

```python
except PromptTooLongError:
    if not state.has_attempted_reactive_compact:
        messages[:] = reactive_compact(messages)
        state.has_attempted_reactive_compact = True
        continue
    return  # Already compacted and still over limit — must exit
```

### 경로 3: 일시적 실패

네트워크 일시 끊김, 429 rate limiting, 529 overload — 이것들은 버그가 아니라 분산 시스템에서는 정상적인 현상이다.

429와 529 모두 exponential backoff + jitter를 사용한다: 첫 시도에서 0.5초, 두 번째에서 1초, 세 번째에서 2초씩 기다리며 최대 10회 재시도한다. 무작위 jitter는 동시 요청들이 같은 순간에 한꺼번에 재시도하는 것을 막아준다. 529 overload 에러가 3회 연속 발생하면 → fallback 모델로 전환한다(`FALLBACK_MODEL_ID` 환경 변수가 설정되어 있는 경우):

```python
def retry_delay(attempt, retry_after=None):
    if retry_after:
        return retry_after
    base = min(500 * (2 ** attempt), 32000) / 1000
    return base + random.uniform(0, base * 0.25)

def with_retry(fn, state, max_retries=10):
    for attempt in range(max_retries):
        try:
            return fn()
        except (RateLimitError, OverloadedError):
            delay = retry_delay(attempt)
            time.sleep(delay)
            if is_overloaded:
                state.consecutive_529 += 1
                if state.consecutive_529 >= 3 and FALLBACK_MODEL:
                    state.current_model = FALLBACK_MODEL
    raise MaxRetriesExceeded()
```

Backoff 공식: `min(500 × 2^attempt, 32000) + random(0~25%)`. 서버가 `Retry-After` 헤더를 반환하면 그 값이 우선한다.

### 모두 합치기

```python
def agent_loop(messages, context):
    system = get_system_prompt(context)
    state = RecoveryState()
    max_tokens = 8000

    while True:
        try:
            response = with_retry(
                lambda: client.messages.create(
                    model=state.current_model, system=system,
                    messages=messages, tools=TOOLS,
                    max_tokens=max_tokens),
                state)
        except Exception as e:
            if is_prompt_too_long_error(e):
                if not state.has_attempted_reactive_compact:
                    messages[:] = reactive_compact(messages)
                    state.has_attempted_reactive_compact = True
                    continue
                return
            log_error(e)
            return

        # max_tokens check BEFORE appending to messages
        if response.stop_reason == "max_tokens":
            if not state.has_escalated:
                max_tokens = 64000
                state.has_escalated = True
                continue  # retry same request, messages unchanged
            # save truncated output + continuation prompt
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": CONTINUATION_PROMPT})
            continue
        # Normal completion
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":
            return
        # ... tool execution ...
```

바깥쪽 try/except는 API 예외(prompt_too_long 등)를 잡고, `with_retry`는 일시적 에러(429/529)를 처리하며, `stop_reason` 검사는 잘림을 처리한다. 세 가지 복구 메커니즘이 각자 자신의 에러 유형을 담당한다.

---

## s10에서 달라진 점

| 구성 요소 | 이전 (s10) | 이후 (s11) |
|-----------|-------------|-------------|
| 에러 처리 | 없음 (어떤 에러든 crash) | 세 가지 복구 패턴 + exponential backoff |
| 새 상수 | — | ESCALATED_MAX_TOKENS=64000, MAX_RETRIES=10, BASE_DELAY_MS=500, FALLBACK_MODEL |
| 새 함수 | — | with_retry, retry_delay, reactive_compact, is_prompt_too_long_error, RecoveryState |
| 도구 | bash, read_file, write_file (3개) | bash, read_file, write_file (3개) — 변경 없음 |
| Loop | 단순 LLM 호출 | try/except + continue 재시도로 감쌈 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s11_error_recovery/code.py
```

다음 prompt들을 시도해보라:

1. Agent에게 아주 긴 코드 한 덩어리를 생성하라고 요청하고, 잘림 이후 자동으로 이어서 작성하는지 관찰해보라(`[max_tokens] escalating` 로그를 확인)
2. 여러 파일을 연속으로 읽어 context를 부풀린 뒤, reactive compact를 관찰해보라
3. 429/529를 만나면 exponential backoff 로그 출력을 관찰해보라

---

## 다음 단계

이제 Agent는 에러로부터 자동으로 복구할 수 있다. 하지만 처리하는 작업은 여전히 일회성이다 — 작업을 주면 끝내고, 그걸로 끝이다.

만약 Agent가 의존성을 가지고, 디스크에 영속화되며, 세션을 넘어 재개 가능한 **task list**를 관리할 수 있다면 어떨까? TODO 목록은 task 시스템이 아니다.

s12 Task System → 작업들이 상태와 영속성을 가진 의존성 그래프를 이룬다. 이것이 multi-Agent 협업의 토대다.

<details>
<summary>CC 소스 코드 Deep Dive</summary>

> 아래 내용은 CC 소스 코드를 기반으로 한다: `query.ts` (1729줄), `services/api/withRetry.ts` (822줄), `query/tokenBudget.ts` (93줄), `utils/tokenBudget.ts` (73줄).

### 1. 십수 개의 Reason/Transition 코드 (단지 3개가 아니다)

교육용 버전은 가장 흔한 3개의 복구 패턴을 다룬다. CC는 실제로 십수 개의 reason/transition 코드를 가지고 있으며, 매 LLM 호출 후마다 평가된다:

| Reason/Transition | 교육용 버전 | CC 동작 |
|---|---|---|
| `completed` | 정상 완료 | 결과 반환 |
| `next_turn` | 정상 tool 호출 | 다음 tool 실행 라운드로 진행 |
| `max_output_tokens_escalate` | 경로 1 | 8K→64K 증액 |
| `max_output_tokens_recovery` | 경로 1 continuation | Continuation prompt (최대 3회) |
| `reactive_compact_retry` | 경로 2 | Reactive compact → 재시도 |
| `prompt_too_long` | 경로 2 | 위와 동일 |
| `collapse_drain_retry` | 미포함 | Context collapse — staged된 콘텐츠를 먼저 commit |
| `model_error` | 미포함 | 재시도 |
| `image_error` | 미포함 | `ImageSizeError` / `ImageResizeError`를 별도로 처리 |
| `aborted_streaming` | 미포함 | Streaming abort 복구 |
| `aborted_tools` | 미포함 | Tool abort |
| `stop_hook_blocking` | 미포함 | Blocking 에러 주입 → 모델이 스스로 교정 |
| `stop_hook_prevented` | 미포함 | Hook이 실행을 막음 |
| `hook_stopped` | 미포함 | Hook이 실행을 중단 |
| `token_budget_continuation` | 미포함 | 토큰 사용량 < 90%일 때 계속 진행 |
| `blocking_limit` | 미포함 | Blocking 한계 도달 |
| `max_turns` | 미포함 | 최대 turn 수 도달 |

교육용 버전은 처음 5개(가장 흔한 것)만 확장해서 다룬다. 나머지는 각각 고유의 전용 처리 로직을 가지고 있다.

### 2. 정확한 Exponential Backoff 공식

CC의 backoff 지연(`withRetry.ts:530-548`):

```
delay = min(500 × 2^(attempt-1), 32000) + random(0~25%)
```

| Attempt | Base Delay | + Jitter |
|---------|-----------|----------|
| 1 | 500ms | 0-125ms |
| 2 | 1000ms | 0-250ms |
| 4 | 4000ms | 0-1000ms |
| 7+ | 32000ms (상한) | 0-8000ms |

서버가 `Retry-After` 헤더를 반환하면 그 값이 우선한다.

### 3. 원본 CONTINUATION Prompt

CC의 continuation prompt(`query.ts:1225-1227`):

```
Output token limit hit. Resume directly — no apology, no recap of what
you were doing. Pick up mid-thought if that is where the cut happened.
Break remaining work into smaller pieces.
```

토큰 예산 nudge prompt(`tokenBudget.ts:72`):

```
Stopped at {pct}% of token target. Keep working — do not summarize.
```

### 4. Streaming 에러 처리

CC의 streaming 경로에서는 복구 가능한 에러(413, max_tokens, media 에러)가 streaming 중에 **화면에 표시되지 않도록 차단된다**(`query.ts:788-822`) — SDK 소비자는 이를 보지 못하고, 복구 로직만 본다. Streaming이 끝난 뒤 시스템이 복구가 필요한지 판단한다.

### 5. 529 → Fallback 모델 전환

529 overload 에러가 3회 연속 발생하면(`MAX_529_RETRIES = 3`), CC는 자동으로 fallback 모델로 전환한다(예: Opus → Sonnet). 전환 시 대기 중인 모든 메시지와 tool 결과가 초기화되며, 사용자에게는 "Switched to {model} due to high demand"가 표시된다.

### 6. 수익 체감(Diminishing Returns) 감지

토큰 예산 "continuation"은 무제한이 아니다. 토큰 증가분이 500 미만인 continuation이 3회 연속 발생하면, 시스템은 "계속해도 의미 있는 output이 나오지 않는다"고 판단하고 continuation을 중단한다(`tokenBudget.ts:60-62`).

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
