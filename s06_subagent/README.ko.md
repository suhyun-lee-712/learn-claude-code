# s06: Subagent — 깨끗한 context로 큰 작업을 작은 작업으로 쪼개기

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → s02 → s03 → s04 → s05 → `s06` → [s07](../s07_skill_loading/) → s08 → ... → s20

> *"큰 작업을 작게 쪼개고, 각각에 깨끗한 context를 부여한다"* — Subagent는 독립적인 messages[]를 사용하므로 메인 대화를 오염시키지 않는다.
>
> **Harness 레이어**: Sub-Agent — Context를 격리하여 주의(attention)가 흩어지지 않게 한다.

---

## 문제

Agent가 버그를 고치고 있다. 호출 체인을 추적하기 위해 30개의 파일을 읽으면서 그 과정에서 60턴에 걸쳐 대화를 주고받는다. messages 리스트는 120개 항목까지 불어나는데, 대부분은 "호출 체인 추적"에서 나온 중간 단계들로, 최종 목표인 "버그 수정"과는 무관하다.

이런 중간 단계들이 context 공간을 차지하면서 Agent는 점점 "건망증"에 걸린다 — 원래 문제가 무엇이었는지조차 기억하지 못하게 된다.

다르게 생각해 보자. 당신이 버그를 고칠 때라면, 호출 체인을 추적하기 위해 "새 터미널을 연다." 다 끝나면 그 터미널을 닫고, 결과를 노트에 기록한 다음, 원래 터미널로 돌아와 계속 고친다. Agent에게도 이 능력이 필요하다 — **독립적인 서브 프로세스를 열고, 독립적인 message 리스트를 주어, 한 가지 일에만 집중하게 하는 것이다.**

---

## 해법

![Subagent 개요](images/subagent-overview.en.svg)

이전 장의 최소 hook 구조와 `todo_write` tool은 그대로 유지하고, 이번 장에서는 새로운 `task` tool에 집중한다. 이 tool이 호출되면 새로운 `messages[]`를 가진 sub-Agent를 생성하여 자체 loop를 돌리고, 메인 Agent에게는 요약 텍스트만 반환한다. 대화 context는 폐기되지만, 파일 시스템 부수 효과(쓰기, 수정, 명령 실행)는 작업 디렉터리에 그대로 남는다.

sub-Agent의 tool은 제한된다. bash/read/write/edit/glob은 가지고 있지만 task는 없어서 재귀적인 생성을 막는다. sub-Agent의 tool 호출도 여전히 permission hook을 거치므로, context 격리가 보안을 우회하지는 않는다.

---

## 작동 방식

**spawn_subagent**는 sub-Agent에게 새로운 messages 리스트를 주고, 자체 loop를 돌린 뒤, 결론만 반환한다:

```python
def spawn_subagent(description: str) -> str:
    # Sub-Agent tools: base tools, but no task (no recursion)
    sub_tools = [...]
    messages = [{"role": "user", "content": description}]  # fresh messages[]

    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUB_SYSTEM,
            messages=messages, tools=sub_tools, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                blocked = trigger_hooks("PreToolUse", block)
                if blocked:
                    results.append({... "content": str(blocked)})
                    continue
                handler = SUB_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown"
                trigger_hooks("PostToolUse", block, output)
                results.append({... "content": output})
        messages.append({"role": "user", "content": results})

    # Return only the final text conclusion, all intermediate steps discarded
    return extract_text(messages[-1]["content"])
```

메인 Agent는 다른 tool과 똑같은 방식으로 이를 호출한다:

```python
TOOLS = [
    {"name": "bash", ...},
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "edit_file", ...},
    {"name": "glob", ...},
    {"name": "todo_write", ...},
    # s06: new task tool
    {"name": "task",
     "description": "Launch a subagent to handle a complex subtask. Returns only the final conclusion.",
     "input_schema": {"type": "object", "properties": {"description": {"type": "string"}}, "required": ["description"]}},
]

TOOL_HANDLERS["task"] = spawn_subagent
```

세 가지 핵심 설계 결정:

| 결정 | 선택 | 이유 |
|----------|--------|--------|
| Context 격리 | 새로운 `messages[]` | sub-Agent의 중간 단계가 메인 Agent의 context를 오염시키지 않는다 |
| 결론만 반환 | `extract_text(last_message)` | 전체 messages 리스트를 반환하지 않는다 |
| 재귀 금지 | sub-Agent에는 task tool이 없다 | sub-Agent가 또 다른 sub-Agent를 생성하는 것을 막는다 |
| 보안 우회 불가 | sub-Agent의 tool 호출은 PreToolUse hook을 거친다 | context 격리가 곧 권한 격리를 의미하지는 않는다 |

디스패치 메커니즘은 변하지 않으며, task tool은 `TOOL_HANDLERS[block.name]`을 통해 라우팅된다. sub-Agent는 자체 `SUB_SYSTEM` prompt를 가지며, 여기서 "작업을 완료하라, 더 이상 위임하지 말라"고 명시적으로 지시한다.

---

## s05에서 달라진 점

| 구성 요소 | 이전 (s05) | 이후 (s06) |
|-----------|-------------|-------------|
| Tool 개수 | 6개 (bash, read, write, edit, glob, todo_write) | 7개 (+task) |
| 새 함수 | — | spawn_subagent (독립적인 messages[] + 30턴 안전 제한) |
| Context 격리 | 모든 것이 메인 대화 안에 있음 | sub-Agent는 새로운 messages[]를 사용 |
| Loop | 변동 없음 | 디스패치는 그대로, sub-Agent는 독립적인 SUB_SYSTEM과 hook으로 보호되는 loop를 가짐 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s06_subagent/code.py
```

다음 prompt들을 시도해 보자:

1. `Use a subtask to find what testing framework this project uses` (sub-Agent가 파일을 읽고, 메인 Agent는 결론만 받는다)
2. `Delegate: read all .py files in agents/ and summarize what each one does`
3. `Use a task to create s06_subagent/example/string_tools.py with a slugify(text: str) function, then verify it from the parent agent`

확인할 점: `[Subagent spawned]` / `[Subagent done]`가 출력되는가? sub-Agent의 tool 호출이 `[sub] ...`로 출력되는가? 부모 Agent가 sub-Agent가 반환한 요약만 가지고 작업을 이어가는가?

---

## 다음 단계

이제 Agent는 작업을 쪼갤 수 있다. 하지만 서로 다른 작업에는 서로 다른 지식이 필요하다. 프론트엔드 컴포넌트를 수정하려면 React 컨벤션이 필요하고, SQL을 작성하려면 테이블 스키마가 필요하다. 이 모든 지식을 system prompt에 욱여넣으면 context가 폭발할 것이다.

→ s07 Skill Loading: 문서를 system prompt에 쌓아 올리는 대신 필요할 때 skill을 주입한다. 필요할 때만, 파일을 읽는 것처럼 자연스럽게 로드한다.

<details>
<summary>Dive into CC Source Code</summary>

> 다음은 CC 소스 코드 `AgentTool.tsx`, `runAgent.ts`, `forkSubagent.ts`, `forkedAgent.ts`에 대한 완전한 분석을 바탕으로 한다.

### 1. 한 가지 패턴이 아니라 세 가지

교육용 버전은 "새로운 messages[]"만 다룬다. 실제 CC에는 세 가지 실행 모드가 있다:

| 모드 | 트리거 | Context |
|------|---------|---------|
| **Normal Subagent** | `subagent_type` 지정 (일반 경로) | 진짜로 새로운 messages[], prompt만 있음 |
| **Fork Subagent** | `subagent_type` 없음, fork gate 활성화 | `buildForkedMessages()`로 캐시 친화적인 prefix를 구성, prompt cache 공유 |
| **General-Purpose** | `subagent_type` 없음, fork gate 비활성화 | Normal과 동일 |

### 2. Fork 모드: Prompt Cache 공유

이것은 교육용 버전이 생략한 핵심 개념이다. Fork 모드(`forkSubagent.ts:60-71`)는 새로운 context를 만들지 않는다. 대신 `buildForkedMessages()`(`forkSubagent.ts:107-168`)를 통해 캐시 친화적인 message prefix를 구성하여, 부모 assistant message를 보존하고 placeholder tool result를 생성한다. 목표는 격리가 아니라, Anthropic API의 prompt cache를 적중시키는 것이다. 부모와 자식 Agent의 system prompt, tools, message prefix가 바이트 단위로 동일하므로 API가 다시 계산할 필요가 없다.

캐시 적중을 위한 다섯 가지 핵심 구성 요소(`forkedAgent.ts:57-68`): system prompt, tools, model, message prefix, thinking config는 반드시 바이트 단위로 동일해야 한다.

### 3. Context 격리의 정밀한 단위

`createSubagentContext()`(`forkedAgent.ts:345-462`)는 sub-Agent의 `ToolUseContext`를 생성한다:

| 필드 | 동작 |
|-------|----------|
| `abortController` | 새로운 자식 controller. 부모의 abort가 아래로 전파된다 |
| `setAppState` | 기본적으로 no-op. 단, 동기 agent는 `shareSetAppState`(`runAgent.ts:697-714`)를 통해 공유한다 |
| `readFileState` | **부모로부터 복제됨** (같은 파일을 다시 읽는 것을 피한다) |
| `queryTracking` | 새로운 chainId, `depth = parentDepth + 1` |

sub-Agent는 완전히 격리되지 않는다. 파일 읽기 상태는 공유된다. UI와 알림의 격리 정도는 실행 경로(sync/async/fork/teammate)에 따라 다르다.

### 4. 재귀적 Fork 보호

교육용 버전은 재귀 보호를 위해 "sub-Agent에는 task tool이 없다"를 사용한다. 실제 구현은 더 미묘하다. `isInForkChild()`(`forkSubagent.ts:78-89`)는 history에서 `FORK_BOILERPLATE_TAG`를 확인한다. 하지만 `constants/tools.ts:36-46`은 기본적으로 `Agent`를 모든 agent의 disabled set에 넣는다(`USER_TYPE === 'ant'` 예외 있음). `forkSubagent.ts:73-89`에는 fork 자식 전용 재귀 보호가 있고, `agentToolUtils.ts:100-110`에는 teammate 시나리오를 위한 특별 허용이 있다. 단순히 "더 이상 sub-Agent를 만들지 않는다"가 아니다.

### 5. Permission 버블링

Fork Agent의 `permissionMode: 'bubble'`(`forkSubagent.ts:67`)은 sub-Agent의 permission prompt가 부모 터미널로 올라온다는 의미다. 즉, 사용자가 메인 터미널에서 sub-Agent의 작업을 승인한다.

### 6. Async vs Sync

교육용 버전은 동기 sub-Agent(부모가 자식이 끝날 때까지 기다림)만 보여준다. CC는 비동기 경로(`AgentTool.tsx:686-764`)도 지원한다. `run_in_background: true`일 때 sub-Agent는 비동기로 실행되어, 즉시 부모에게 `{ status: 'async_launched' }`를 반환하고, 완료되면 부모에게 알린다. 실제 트리거는 `run_in_background`를 넘어서, auto-background, assistant force async, coordinator/proactive 경로까지 포함한다.

### 교육용 버전의 단순화는 의도적이다

- 세 가지 모드 → 하나(새로운 messages): 개념적으로 명확함
- Prompt cache 공유 → 생략: 교육용 버전은 API 레이어 최적화를 다루지 않음
- 재귀적 fork 보호 → "sub-Agent에는 task tool이 없다"로 단순화
- Async → 생략 (s13으로 미룸): s06은 동기 모델을 먼저 다룸

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
