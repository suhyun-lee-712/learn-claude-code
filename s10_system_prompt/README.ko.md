# s10: System Prompt — 런타임에 조립되며, 절대 하드코딩하지 않는다

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s08 → s09 → `s10` → [s11](../s11_error_recovery/) → s12 → ... → s20
> *"prompt is assembled, not hardcoded"* — 섹션 + 온디맨드 조립 + 캐싱.
>
> **Harness Layer**: Prompt — 런타임에 조립되며, 절대 하드코딩하지 않는다.

---

## 문제

s01부터 s09까지, system prompt는 항상 하드코딩된 한 줄이었습니다:

```python
SYSTEM = f"You are a coding agent at {WORKDIR}. Use tools to solve tasks."
```

s01에서는 이걸로 충분했습니다 — bash, read, write뿐이었으니까요. 하지만 s09에 이르러 Agent는 memory, 압축(compression), skill 로딩 기능을 갖게 됩니다. prompt는 점점 더 많은 기능을 설명해야 합니다:

```python
SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Use tools to solve tasks. Act, don't explain. "
    "Before starting any multi-step task, use todo_write. "
    "Skills are available via list_skills and load_skill. "
    "Relevant memories are injected below when available. "
    # ... add a capability, add a line
)
```

여기엔 세 가지 문제가 있습니다:

1. **프로젝트를 전환하려면 prompt 전체를 다시 작성해야 한다** — 무엇을 바꾸고 무엇을 유지해야 할지 알 방법이 없다
2. **하나를 바꾸면 다른 것이 깨질 수 있다** — tool 설명을 추가하면 앞쪽 지시사항과 충돌할 수 있다
3. **모든 요청이 모든 것을 짊어진다** — 현재 대화에서 특정 섹션이 필요 없을 때조차 토큰을 낭비한다

system prompt는 현재 상태를 기반으로 런타임에 조립되는 설정(configuration)이어야 합니다: 어떤 tool이 활성화되어 있는지, 어떤 context가 보이는지, 어떤 memory가 관련 있는지, 그리고 prompt cache를 적중시키기 위해 어떤 내용이 안정적으로 유지되어야 하는지 말이죠.

---

## 해법

![System Prompt 개요](images/system-prompt-overview.en.svg)

s10은 prompt 조립에 집중합니다. s08-s09의 기능들 위에 쌓아 올리되, 압축이나 memory를 다시 구현하지는 않습니다. 핵심 변경은 다음과 같습니다: 하드코딩된 `SYSTEM`을 독립적인 섹션들로 분리하고, 실제 상태를 기반으로 런타임에 조립한 뒤, 그 결과를 캐싱합니다.

네 개의 섹션, 두 가지 로딩 전략:

| Section | Strategy | Content | Condition |
|---------|----------|---------|-----------|
| identity | always | 당신이 누구인지, 어떻게 일하는지 | 항상 존재 |
| tools | always | 사용 가능한 tool 목록 | `enabled_tools` |
| workspace | always | 작업 디렉터리 | 항상 존재 |
| memory | on-demand | 관련 memory 내용 | `.memory/MEMORY.md` 존재 여부 |

핵심 설계: 섹션이 로드될지 여부는 실제 상태(tool이 존재하는지, 파일이 존재하는지)에 따라 결정되며, 메시지 안의 키워드에 따라 결정되지 않습니다.

---

## 동작 방식

### PROMPT_SECTIONS: 토픽을 키로 갖는 조각들

거대한 단일 문자열을 딕셔너리로 분리하고, 각 키는 하나의 토픽이 됩니다:

```python
PROMPT_SECTIONS = {
    "identity": "You are a coding agent. Act, don't explain.",
    "tools": "Available tools: bash, read_file, write_file.",
    "workspace": f"Working directory: {WORKDIR}",
    "memory": "Relevant memories are injected below when available.",
}
```

각 섹션은 독립적으로 관리됩니다. `tools`를 바꿔도 `identity`에 영향을 주지 않고, `memory`를 추가해도 `workspace`를 건드리지 않습니다.

### assemble_system_prompt: 온디맨드 조립

모든 턴마다 모든 섹션이 필요한 것은 아닙니다. memory 파일이 없나요? 그렇다면 memory 섹션을 로드하는 것은 토큰 낭비일 뿐입니다. 조립은 context의 실제 상태를 기반으로 이루어집니다:

```python
def assemble_system_prompt(context: dict) -> str:
    sections = []

    # Always loaded
    sections.append(PROMPT_SECTIONS["identity"])
    sections.append(PROMPT_SECTIONS["tools"])
    sections.append(PROMPT_SECTIONS["workspace"])

    # On-demand — based on real state, not keywords
    memories = context.get("memories", "")
    if memories:
        sections.append(f"Relevant memories:\n{memories}")

    return "\n\n".join(sections)
```

"Always loaded" 섹션은 매 턴마다 필요합니다: identity, tools, workspace. "On-demand" 섹션은 특정 조건에서만 유용합니다.

왜 전부 로드하지 않을까요? 토큰에는 비용이 있고(system prompt는 매 턴마다 과금됩니다), 지시사항이 적을수록 출력이 더 집중됩니다(관련 없는 지시사항은 노이즈일 뿐입니다).

### get_system_prompt: 재조립을 피하기 위한 캐싱

context가 바뀌지 않았을 때(같은 턴 안에서 동일한 context로 LLM을 여러 번 호출하는 경우), 다시 조립하는 것은 낭비입니다. 결정적(deterministic) 직렬화를 사용해 변경을 감지하고 캐싱된 결과를 반환합니다:

```python
def get_system_prompt(context: dict) -> str:
    global _last_context_key, _last_prompt
    key = json.dumps(context, sort_keys=True, ensure_ascii=False, default=str)
    if key == _last_context_key and _last_prompt:
        return _last_prompt
    _last_context_key = key
    _last_prompt = assemble_system_prompt(context)
    return _last_prompt
```

`hash()` 대신 `json.dumps`를 쓰는 이유: 파이썬의 내장 `hash()`는 프로세스 무작위화(randomization)가 적용되어 있어(안정적인 캐시 키로는 부적합) 중첩된 dict/list에 대해 `unhashable type` 오류를 던집니다.

참고: 이 캐시는 한 프로세스 내에서 중복된 문자열 조립을 피하는 것일 뿐입니다. 이것은 CC의 API prompt cache와는 다릅니다. CC는 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`를 사용해 정적(static) 부분과 동적(dynamic) 부분을 분리하는데, 정적 부분은 글로벌 캐시에 적중하며 동적 내용이 바뀌어도 무효화되지 않습니다.

### context: 키워드 추측이 아닌 실제 상태

context는 실제 런타임 상태를 반영합니다:

```python
def update_context(context: dict, messages: list) -> dict:
    memories = ""
    if MEMORY_INDEX.exists():
        content = MEMORY_INDEX.read_text().strip()
        if content:
            memories = content
    return {
        "enabled_tools": list(TOOL_HANDLERS.keys()),
        "workspace": str(WORKDIR),
        "memories": memories,
    }
```

`enabled_tools`는 실제로 등록된 tool들을 나열합니다. `memories`는 `.memory/MEMORY.md`가 존재하는지 확인합니다. 섹션 로딩은 이 실제 상태를 기반으로 하며, 메시지에서 키워드를 검색하는 방식이 아닙니다.

### 한데 모으기

```python
def agent_loop(messages: list, context: dict):
    system = get_system_prompt(context)
    while True:
        response = client.messages.create(
            model=MODEL, system=system, messages=messages,
            tools=TOOLS, max_tokens=8000)
        # ... tool execution ...
        context = update_context(context, messages)
        system = get_system_prompt(context)
```

loop 반복의 시작마다 system prompt를 가져옵니다. context가 바뀌었으면 다시 조립하고, 바뀌지 않았으면 캐싱된 버전을 반환합니다.

---

## s09로부터의 변경점

| Component | Before (s09) | After (s10) |
|-----------|-------------|-------------|
| prompt | 하드코딩된 SYSTEM 문자열 | PROMPT_SECTIONS + assemble_system_prompt |
| caching | 없음 | get_system_prompt (json.dumps 감지 + 캐시) |
| new functions | — | assemble_system_prompt, get_system_prompt, update_context |
| tools | bash, read_file, write_file (3개) | bash, read_file, write_file (3개) — 변경 없음 |
| loop | 고정된 SYSTEM 사용 | get_system_prompt(context) 사용 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s10_system_prompt/code.py
```

지켜볼 것:

1. 출력에 어떤 섹션이 로드되었는지 표시됩니다 (`[assembled] sections: ...` 라벨)
2. 대화가 이어지는 동안 캐시 적중 시 `[cache hit]`이 표시됩니다
3. `.memory/MEMORY.md`를 생성하면 다음 턴에 memory 섹션이 나타납니다

다음 prompt들을 시도해 보세요:

1. `Read the file README.md` (항상 로드되는 세 섹션을 관찰)
2. `Create a file called .memory/MEMORY.md with content "- [test](test.md) — test memory"` (memory 인덱스 작성)
3. `Read the file code.py` (memory 섹션이 나타나는지 관찰)

---

## 다음 단계

이제 system prompt를 런타임에 조립할 수 있게 되었습니다. 하지만 Agent는 여전히 에러가 나면 크래시합니다. 네트워크 끊김, API rate limit, 잘린 출력, context 오버플로 — 이것들은 버그가 아니라 정상적인 일입니다.

s11 Error Recovery → 네 가지 복구 경로. 토큰 늘리기, context 압축, 지수 백오프(exponential backoff), 모델 전환.

<details>
<summary>CC 소스 코드 깊이 들여다보기</summary>

> 아래 내용은 CC 소스 코드 `constants/prompts.ts` (914줄), `constants/systemPromptSections.ts` (68줄), `context.ts` (189줄), `utils/api.ts` (718줄), `utils/systemPrompt.ts` (123줄), 그리고 `bootstrap/state.ts`에 대한 분석을 기반으로 합니다.

### CC의 system prompt에는 섹션이 몇 개나 있을까?

그 개수는 feature flag, output style, KAIROS/Proactive 모드, 사용자 유형, 토큰 예산 등에 따라 달라집니다. 대략 두 부류로 나뉩니다:

**Static 섹션** (항상 로드): identity, system, doing_tasks, actions, using_tools, tone_style, output_efficiency 등.

**Dynamic 섹션** (상태에 따라 로드): session_guidance, memory, ant_model_override, env_info_simple, language, output_style, mcp_instructions, scratchpad, frc, summarize_tool_results, numeric_length_anchors, token_budget, brief 등.

`mcp_instructions`는 유일하게 휘발성(volatile)인 섹션입니다(`DANGEROUS_uncachedSystemPromptSection()`을 통해 생성됨). MCP 서버는 턴 사이에 연결되거나 끊길 수 있기 때문입니다.

### 조립 함수

```typescript
getSystemPrompt(tools, model, additionalWorkingDirs?, mcpClients?): Promise<string[]>
```

`string[]`을 반환하며(각 요소가 하나의 섹션), 정적 부분과 동적 부분 사이는 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`로 구분됩니다.

### cache scope

글로벌 캐시 경계(boundary)가 활성화되면, 정적 섹션들은 하나의 글로벌 캐시 블록으로 병합되고, 동적 섹션은 글로벌 캐시를 사용하지 않습니다(`cacheScope: null`). 경계가 없거나 글로벌 캐시를 건너뛰는 경로만 org scope로 폴백합니다.

교육용 버전의 캐시는 중복된 문자열 조립을 피하는 것뿐입니다. CC의 세 단계(three-layer) 캐시:

1. **lodash memoize**: `getSystemContext`와 `getUserContext`가 세션 단위로 캐싱됨 (`context.ts`)
2. **Section registry cache**: `STATE.systemPromptSectionCache`가 동적 섹션 결과를 캐싱하며, `/clear`나 `/compact` 시 지워짐
3. **API-level cache**: `splitSysPromptPrefix()` (`api.ts`)가 boundary를 통해 prompt를 서로 다른 cache scope를 가진 블록들로 분할함

### getUserContext vs getSystemContext

| | getSystemContext | getUserContext |
|---|---|---|
| Content | gitStatus, cacheBreaker | CLAUDE.md 내용, currentDate |
| Injection | system prompt 배열에 append | `<system-reminder>` user 메시지로 prepend |
| When skipped | custom system prompt 사용 시 | 항상 실행됨 |

### 모드가 prompt를 바꾸는 방식

- **CLAUDE_CODE_SIMPLE**: prompt 전체가 2줄
- **Proactive/KAIROS**: 간결한(compact) prompt가 모든 표준 섹션을 대체함
- **Coordinator**: coordinator 전용 prompt가 기본값을 완전히 대체함
- **Agent mode**: agent가 정의한 prompt가 기본값을 대체하거나 거기에 append함

### 전체 크기

표준 대화형(interactive) 모드의 system prompt 코어는 약 20-30KB 텍스트입니다. CLAUDE_CODE_SIMPLE은 약 150자입니다. user context(CLAUDE.md)와 system context(git status)가 그 위에 더해집니다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
