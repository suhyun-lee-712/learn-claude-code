# s08: Context Compact — 컨텍스트는 가득 차기 마련, 공간을 비울 방법을 마련하라

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → s02 → s03 → s04 → s05 → s06 → s07 → `s08` → [s09](../s09_memory/) → s10 → ... → s20
> *"컨텍스트는 가득 차기 마련 — 공간을 비울 방법을 마련하라"* — 4계층 압축 파이프라인: 싼 것부터 먼저, 비싼 것은 나중에.
>
> **Harness 계층**: 압축 — 깨끗한 메모리, 무제한 세션.

---

## 문제

Agent가 잘 돌아가다가 갑자기 멈춘다.

bash, read, write 등 필요한 모든 기능을 갖추고 있다. 하지만 1000줄짜리 파일(~4000 토큰)을 읽고, 이어서 파일 30개를 더 읽고, 명령 20개를 실행했다. 모든 명령의 출력, 모든 파일의 내용이 전부 `messages` 리스트에 쌓인다.

context window는 유한하다. 일단 가득 차면 API는 호출 자체를 거부한다: `prompt_too_long`.

압축이 없으면 Agent는 대규모 프로젝트에서 아예 작업을 할 수 없다.

---

## 해결책

![Compact 개요](images/compact-overview.en.svg)

s07의 hook 구조, skill 로딩, sub-Agent는 그대로 유지하되, compaction에 집중하기 위해 일부 tool은 생략했다. 핵심 변경 사항은 다음과 같다. 매 LLM 호출 전에 세 개의 전처리기(API 호출 0회)를 끼워 넣고, 그래도 토큰이 임계값을 초과하면 LLM 요약(API 호출 1회)을 트리거하며, API가 에러를 던지면 긴급 트리밍을 수행한다.

핵심 설계 원칙: 싼 것부터 먼저, 비싼 것은 나중에.

---

## 동작 방식

![4계층 압축 파이프라인](images/compaction-layers.en.svg)

### L1: snip_compact — 관련 없는 오래된 대화 잘라내기

Agent가 80턴의 대화를 진행하면서 `messages` 160개가 쌓였다. 맨 처음 "hello.py 만들어줘"는 현재 작업과 거의 관련이 없는데도 여전히 공간을 차지하고 있다.

메시지 수가 50을 초과하면 → 앞쪽 3개(초기 컨텍스트)와 뒤쪽 47개(현재 작업)는 남기고 중간을 잘라낸다:

```python
def snip_compact(messages, max_messages=50):
    if len(messages) <= max_messages:
        return messages
    keep_head, keep_tail = 3, max_messages - 3
    snipped = len(messages) - keep_head - keep_tail
    placeholder = {"role": "user",
                   "content": f"[snipped {snipped} messages from conversation middle]"}
    return messages[:keep_head] + [placeholder] + messages[-keep_tail:]
```

메시지 전체는 잘려 나가지만, 남아 있는 메시지 안의 `tool_result` 내용은 계속 쌓인다 — 34번 메시지가 여전히 30KB짜리 오래된 파일 내용을 담고 있을 수 있다. → L2.

### L2: micro_compact — 오래된 tool result를 placeholder로 대체

![오래된 result placeholder](images/micro-compact.en.svg)

Agent가 파일 10개를 연속으로 읽었다. 1~7번째로 읽은 파일의 전체 내용은 더 이상 필요 없는데도 여전히 컨텍스트에 남아 막대한 공간을 차지하고 있다.

가장 최근 `tool_result` 3개만 온전히 남기고, 더 오래된 것은 한 줄짜리 placeholder로 대체한다:

```python
KEEP_RECENT_TOOL_RESULTS = 3

def micro_compact(messages):
    tool_results = collect_tool_result_blocks(messages)
    if len(tool_results) <= KEEP_RECENT_TOOL_RESULTS:
        return messages
    for _, _, block in tool_results[:-KEEP_RECENT_TOOL_RESULTS]:
        if len(block.get("content", "")) > 120:
            block["content"] = "[Earlier tool result compacted. Re-run if needed.]"
    return messages
```

오래된 result는 정리되지만, 새 result 하나가 500KB일 수도 있다 — 큰 파일을 `cat` 한 번 하는 것만으로 컨텍스트가 가득 찰 수 있다. → L3.

### L3: tool_result_budget — 큰 result를 디스크에 영속화

![큰 result를 디스크로](images/layer1-budget.en.svg)

모델이 한 번에 큰 파일 5개를 읽었다. 마지막 user 메시지의 모든 `tool_result` 블록 합계가 500KB에 달한다.

마지막 user 메시지의 모든 `tool_result` 블록 크기를 합산한다. 200KB를 초과하면 → 크기순으로 정렬해 가장 큰 것부터 `.task_outputs/tool-results/`에 영속화하고, 컨텍스트에는 `<persisted-output>` 마커와 2000자 미리보기만 남긴다. 모델은 이 마커를 보고 전체 내용이 디스크에 있음을 알며, 필요할 때 다시 읽는다.

```python
def tool_result_budget(messages, max_bytes=200_000):
    last = messages[-1]
    blocks = [(i, b) for i, b in enumerate(last["content"])
              if b.get("type") == "tool_result"]
    total = sum(len(str(b.get("content", ""))) for _, b in blocks)
    if total <= max_bytes:
        return messages
    ranked = sorted(blocks, key=lambda p: len(str(p[1].get("content", ""))), reverse=True)
    for idx, block in ranked:
        if total <= max_bytes:
            break
        block["content"] = persist_large_output(block["tool_use_id"], str(block["content"]))
        total = recalculate_total(blocks)
    return messages
```

앞의 세 계층은 모두 평문/구조적 연산이라 — API 호출 0회 — 대화 내용을 "이해"하지는 못한다. 컨텍스트가 여전히 너무 클 수 있다. → L4.

### L4: compact_history — 전체 LLM 요약

![전체 LLM 요약](images/auto-compact.en.svg)

앞의 세 계층이 모두 실행되었지만, 거대한 프로젝트에서 30분간 쉬지 않고 작업한 결과 토큰이 여전히 임계값을 초과한다.

3단계 프로세스:

1. **transcript 저장**: 전체 대화를 JSONL 형식으로 `.transcripts/`에 기록한다. transcript는 복구 가능한 기록을 보존하지만, 모델의 활성 컨텍스트에는 요약만 남는다. 모델의 현재 추론 입장에서 세부 내용은 더 이상 컨텍스트에 없다. 교육용 코드는 transcript를 가져오는 tool을 제공하지 않는다.
2. **LLM이 요약 생성**: 대화 이력을 LLM에 보내, 핵심 정보를 보존하도록 요청한다: 현재 목표, 중요한 발견, 수정된 파일, 남은 작업, 사용자 제약 등.
3. **메시지 리스트 교체**: 오래된 메시지 전부를 하나의 요약으로 대체한다. 교육용 버전은 요약만 남기지만, 실제 Claude Code는 compaction 이후 최근 파일 일부, plan, agent/skill/tool 컨텍스트를 다시 붙인다.

```python
def compact_history(messages):
    transcript_path = write_transcript(messages)  # Save full conversation first
    summary = summarize_history(messages)          # LLM generates summary
    return [{"role": "user",
             "content": f"[Compacted]\n\n{summary}"}]
```

**Circuit breaker**: 3회 연속 실패하면 재시도를 멈춰, 무한 루프가 API 호출을 낭비하는 것을 방지한다.

### Reactive: reactive_compact

때로는 API가 여전히 `prompt_too_long`(413)을 반환한다 — 컨텍스트가 압축 트리거보다 빠르게 커질 때다.

이때 **reactive_compact**가 트리거된다: compact_history보다 더 공격적으로, 꼬리에서부터 후퇴하며 바이트 단위 정밀도로 API가 받아들일 수 있는 크기까지 트리밍하고, 마지막 5개 메시지 + 요약만 남긴다.

```python
def reactive_compact(messages):
    transcript = write_transcript(messages)
    summary = summarize_history(messages)
    tail = messages[-5:]
    return [{"role": "user",
             "content": f"[Reactive compact]\n\n{summary}"}, *tail]
```

reactive compact에는 재시도 제한(기본 1회)이 있다. 그래도 실패하면 영원히 루프를 도는 대신 예외를 던진다. 완전한 에러 복구는 s11로 미룬다.

### 전부 합치기

```python
def agent_loop(messages):
    reactive_retries = 0
    while True:
        # Three pre-processors (0 API calls)
        # Order: budget first, so large content is persisted before placeholders
        messages[:] = tool_result_budget(messages)    # L3: persist large results
        messages[:] = snip_compact(messages)          # L1: trim middle
        messages[:] = micro_compact(messages)         # L2: old result placeholders

        # Still too much? LLM summary (1 API call)
        if estimate_token_count(messages) > THRESHOLD:
            messages[:] = compact_history(messages)

        try:
            response = client.messages.create(...)
        except PromptTooLongError:
            if reactive_retries < MAX_REACTIVE_RETRIES:
                messages[:] = reactive_compact(messages)  # Emergency
                reactive_retries += 1
                continue
            raise  # retry limit exceeded, raise exception
        # ... tool execution ...

        # compact tool: when the model actively calls it, triggers compact_history
        if block.name == "compact":
            messages[:] = compact_history(messages)
            results.append({..., "content": "[Compacted. History summarized.]"})
            messages.append({"role": "user", "content": results})
            break  # end current turn, start fresh with compacted context
```

**순서를 바꿔서는 안 된다.** L3(budget)는 L2(micro)보다 먼저 실행되는데, micro가 오래된 큰 tool_result를 한 줄짜리 placeholder로 대체하기 때문이다 — budget은 그 일이 일어나기 전에 전체 내용을 영속화해야 한다. 이것이 CC 소스가 `applyToolResultBudget`을 가장 먼저 두는 이유다.

---

## s07로부터의 변경 사항

| 구성 요소 | 이전 (s07) | 이후 (s08) |
|-----------|-------------|-------------|
| 컨텍스트 관리 | 없음 (컨텍스트가 무한히 증가) | 4계층 압축 파이프라인 + 긴급 처리 |
| 신규 함수 | — | snip_compact, micro_compact, tool_result_budget, compact_history, reactive_compact |
| Tool | bash, read_file, write_file, edit_file, glob, todo_write, task, load_skill (8) | 8 + compact (9) |
| Loop | LLM 호출 → tool 실행 | 매 턴 전 세 개의 전처리기 + 임계값 트리거 compact_history |
| 설계 원칙 | — | 싼 것부터 먼저, 비싼 것은 나중에 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s08_context_compact/code.py
```

다음 프롬프트를 시도해 보라:

1. `Read the file README.md, then read code.py, then read s01_agent_loop/README.md` (여러 파일을 연속으로 읽어, L2가 오래된 result를 압축하는 것을 관찰)
2. `Read every file in s08_context_compact/` (한 번에 많은 내용을 읽어, L3가 디스크에 영속화하는 것을 관찰)
3. 20턴 이상 대화하며 `[auto compact]` 또는 `[reactive compact]`가 나타나는지 관찰

관찰 포인트: 매 tool 실행 후, 오래된 `tool_result`가 압축되는가? 장시간 대화 끝에 토큰이 임계값을 넘으면 요약이 자동으로 트리거되는가?

---

## 다음 단계

컨텍스트 압축 덕분에 Agent는 멈추지 않고 오래 실행될 수 있다. 하지만 압축할 때마다 사용자가 알려준 선호와 제약도 함께 사라진다. Agent가 중요한 것을 선택적으로 기억하게 할 수 있을까?

s09 Memory → 세 가지 서브시스템: 무엇을 기억할지 선택하기, 핵심 정보 추출하기, 통합하고 정리하기. 압축을 넘어, 세션을 넘어.

<details>
<summary>CC 소스 코드 깊이 파보기</summary>

> 아래 내용은 CC 소스 코드 `compact.ts`, `autoCompact.ts`, `microCompact.ts`, `query.ts` 분석에 기반한다.

### 실행 순서 비교

교육용 버전은 교육적 명확성을 위해 계층에 L1/L2/L3/L4 레이블을 붙였지만, 실제 실행 순서는 번호와 일치하지 않는다:

| 항목 | 교육용 버전 | Claude Code |
|-----------|-----------------|-------------|
| 실행 순서 | budget → snip → micro → auto | budget → snip → micro → collapse → auto (`query.ts:379-468`) |
| snip_compact | head 3 + tail 47 유지 | CC는 메인 스레드에서만 활성화하며, 구현은 오픈소스 레포에 없음 (`HISTORY_SNIP` feature gate). 다만 인터페이스는 보임: `snipCompactIfNeeded(messages)` → `{ messages, tokensFreed, boundaryMessage? }`, 모델이 직접 snip하는 `SnipTool`도 노출. 교육용 버전의 3/47은 단순화된 파라미터 |
| micro_compact | 텍스트 placeholder 대체 | 두 가지 경로: time-based는 내용을 직접 지우고, cached는 API `cache_edits` 사용 (legacy 경로는 제거됨) |
| micro_compact 화이트리스트 | 위치 기준 (가장 최근 3개) | time-based는 시간 임계값으로 트리거, cached는 개수로 트리거 (`microCompact.ts`) |
| tool_result_budget | 200KB 문자 | 200,000 문자 (`toolLimits.ts:49`) |
| compact_history 임계값 | 문자 수 추정 | 정밀 토큰: `contextWindow - maxOutputTokens - 13_000` |
| 요약 요구 사항 | 5개 범주의 정보 | 9개 섹션 + `<analysis>`/`<summary>` 이중 태그 |
| 압축 prompt | 간단한 prompt | tool 호출을 금지하는 양끝단 하드 가드레일 |
| PTL 재시도 | 있음 (단순화) | `truncateHeadForPTLRetry()`가 메시지 그룹 단위로 후퇴 (`compact.ts:243-290`) |
| 압축 후 복구 | 없음 (교육용 버전은 요약만 유지) | 최근 파일, plan, agent/skill/tool 컨텍스트 자동 재읽기 |
| Circuit breaker | 3회 | 3회 (`autoCompact.ts:70`) |
| Reactive 재시도 | 1회 | CC는 더 세분화된 단계별 재시도 보유 |

### 실행 순서 상세

CC 소스 `query.ts`의 실제 순서:

1. `applyToolResultBudget` (L379): 큰 result를 먼저 영속화해 전체 내용이 저장되도록 보장
2. `snipCompact` (L403): 중간 메시지 잘라내기
3. `microcompact` (L414): 오래된 result placeholder 처리
4. `contextCollapse` (L441): 독립적인 컨텍스트 관리 시스템 (교육용 버전에 없음)
5. `autoCompact` (L454): LLM 전체 요약

교육용 버전의 budget → snip → micro 순서는 이와 일치한다. 교육용 버전에는 contextCollapse 메커니즘이 없다.

### read_file 트레이드오프

교육용 버전의 `micro_compact`는 `read_file`을 포함해 오래된 `tool_result` 블록을 일률적으로 placeholder로 대체한다. 이는 보통 기능적 정확성에 영향을 주지 않는다: 나중에 모델이 파일 내용이 필요하면 파일을 다시 읽으면 된다. 비용은 추가 tool 호출과 잠재적으로 낮아지는 prompt cache 적중률이다.

Claude Code는 이를 교육용 버전의 단순한 규칙으로 해결하지 않는다. CC도 `Read`를 microcompact 대상 tool 집합에 넣지만, 별도의 `readFileState`를 유지한다: 변경되지 않은 파일을 반복해서 읽으면 `FILE_UNCHANGED_STUB`을 반환하고, compaction 이후에는 예산 범위 내에서 최근 읽은 파일 내용을 복원한다 (예를 들어 최대 5개 파일, 파일당 5K 토큰, 총 50K 토큰). 이는 프로덕션 수준의 캐시 및 복구 메커니즘이다. 교육용 버전은 그런 장치까지 확장하지 않고, 오래된 result를 압축하고 필요할 때 다시 읽는 더 단순한 트레이드오프를 유지한다.

### 전체 상수 참조

| 상수 | 값 | 소스 파일 |
|----------|-------|-------------|
| `AUTOCOMPACT_BUFFER_TOKENS` | 13,000 | `autoCompact.ts:62` |
| `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES` | 3 | `autoCompact.ts:70` |
| `MAX_OUTPUT_TOKENS_FOR_SUMMARY` | 20,000 | `autoCompact.ts:30` |
| `POST_COMPACT_TOKEN_BUDGET` | 50,000 | `compact.ts:123` |
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | `compact.ts:122` |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5,000 | `compact.ts:124` |
| Time micro_compact 간격 | 60분 | `timeBasedMCConfig.ts` |
| `MAX_COMPACT_STREAMING_RETRIES` | 2 | `compact.ts:131` |

### contextCollapse와 sessionMemoryCompact

CC 소스 코드에는 이 교육용 버전에서 다루지 않는 두 가지 추가 메커니즘이 있다:

- **contextCollapse**: 독립적인 컨텍스트 관리 시스템으로, 활성화되면 능동적 autocompact를 억제하고 (`autoCompact.ts:215-222`), collapse의 commit/blocking 흐름이 컨텍스트 관리를 넘겨받는다. 수동 `/compact`와 reactive fallback은 독립적인 경로로 남아 contextCollapse의 영향을 받지 않는다.
- **sessionMemoryCompact**: compact_history 전에, CC는 LLM을 호출하지 않고 기존 세션 메모리(s09에서 다룸)를 사용해 가벼운 요약을 먼저 시도한다. 이 메커니즘은 s09를 배운 뒤에 더 명확해진다.

### 압축 prompt는 어떻게 생겼을까?

CC의 압축 prompt에는 두 가지 하드 요구 사항이 있다:

1. **절대 tool을 호출하지 말 것**: `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.`로 시작하고, 끝에 또 하나의 REMINDER를 덧붙인다
2. **먼저 분석하고, 그다음 요약**: 모델은 먼저 `<analysis>` 태그 안에서 추론한 뒤, `<summary>` 태그 안에 정식 요약을 출력해야 한다. analysis는 포매팅 과정에서 제거된다

### 교육용 버전의 단순화는 의도된 것이다

- micro_compact가 텍스트 placeholder를 사용함 → API 수준의 `cache_edits`에 접근할 수 없다
- read_file을 특별 취급하지 않음 → 교육용 버전은 readFileState와 compaction 후 복구를 도입하는 대신 필요할 때 다시 읽는 것을 받아들인다
- 문자 수로 토큰을 추정함 → 정밀 tokenizer는 범위 밖이다
- compaction 후 복구를 생략함 → 교육용 버전은 요약만 남기고 파일을 자동으로 다시 붙이지 않는다
- 두 가지 보조 메커니즘을 다루지 않음 → 이들은 10% 디테일 범주에 속한다

핵심 설계 원칙인 "싼 것부터 먼저, 비싼 것은 나중에"는 온전히 보존되어 있다.

</details>

<!-- translation-sync: zh@v2, en@v2, ja@v2 -->
