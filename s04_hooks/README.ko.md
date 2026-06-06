# s04: Hooks — loop에 끼워 넣지 말고, loop에 매달아라

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → s02 → s03 → `s04` → [s05](../s05_todo_write/) → s06 → ... → s20

> *"loop에 끼워 넣지 말고, loop에 매달아라"* — Hook은 tool 실행 전후에 확장 로직을 주입한다.
>
> **Harness Layer**: Hooks — loop를 침범하지 않는 확장 지점.

---

## 문제

s03의 Agent에는 권한 체크가 있다. 하지만 "모든 bash 호출을 로그로 남겨라", "write 이후 자동으로 git add 하라" 같은 새 체크를 추가할 때마다 `agent_loop` 함수를 수정해야 한다.

loop는 금세 이런 모습이 된다:

```python
def agent_loop(messages):
    while True:
        # ... LLM call ...
        for block in response.content:
            if block.type != "tool_use":
                continue
            log_to_file(block)          # added a line
            check_permission(block)     # added a line
            notify_slack(block)         # added another line
            output = execute(block)
            auto_git_add(block)         # yet another line
            # ... the loop is unrecognizable
```

확장하고 싶은 것은 Agent의 동작인데, 정작 수정하는 것은 loop 그 자체다. loop는 안정적인 핵심으로 남아야 하고, 확장은 바깥에 매달려야 한다.

---

## 해결책

![Hooks 개요](images/hooks-overview.en.svg)

s03의 loop와 권한 로직은 그대로 완전히 보존된다. 유일한 변경은 `check_permission()`을 loop 본문 안에서 hook 위로 옮긴 것뿐이다. 이제 loop는 어떤 체크 함수도 직접 호출하지 않는다. 대신 `trigger_hooks("PreToolUse", block)`을 호출하고, 무엇을 실행할지는 registry가 결정한다.

네 개의 이벤트가 완전한 agent 사이클을 커버한다:

| 이벤트 | 트리거 시점 | 대표적 용도 |
|-------|---------------|-------------|
| UserPromptSubmit | 사용자 입력 후, LLM 진입 전 | 입력 검증, context 주입 |
| PreToolUse | tool 실행 전 | 권한 체크, 로깅 |
| PostToolUse | tool 실행 후 | 부수 효과(자동 git add 등), 출력 체크 |
| Stop | loop가 막 종료되려 할 때 | 정리(CC는 강제 continuation도 지원) |

확장은 `register_hook()`으로 추가한다. loop는 `trigger_hooks()`만 호출한다.

---

## 동작 원리

**Hook registry**: 이벤트 이름을 콜백 리스트에 매핑하는 dict.

```python
HOOKS = {
    "UserPromptSubmit": [],
    "PreToolUse": [],
    "PostToolUse": [],
    "Stop": [],
}

def register_hook(event: str, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:   # return value ≠ None → hook says "stop"
            return result
    return None
```

교육용 버전에서 PreToolUse가 non-None을 반환하면 실행을 차단한다는 뜻이고, Stop이 non-None을 반환하면 강제 continuation을 의미한다. UserPromptSubmit과 PostToolUse의 반환값은 사용되지 않는다.

**UserPromptSubmit**, 사용자 입력 후 LLM 진입 전에 트리거된다. CC는 입력을 가로채거나 수정할 수 있지만, 교육용 버전은 로그만 남긴다:

```python
def context_inject_hook(query: str) -> str | None:
    """Inject current working directory info into every prompt."""
    print(f"\033[90m[HOOK] UserPromptSubmit: working in {WORKDIR}\033[0m")
    return None   # return None = no modification, let prompt through

register_hook("UserPromptSubmit", context_inject_hook)
```

메인 loop에서는 사용자 입력 직후에 트리거된다:

```python
query = input("s04 >> ")
trigger_hooks("UserPromptSubmit", query)   # ← before entering LLM
history.append({"role": "user", "content": query})
agent_loop(history)
```

**PreToolUse / PostToolUse**, tool 실행 전후의 hook이다. s03의 권한 체크 로직은 이제 PreToolUse hook으로 감싸졌고, 여기에 로깅 hook과 대용량 출력 알림이 더해졌다:

```python
# PreToolUse: permission check (s03 logic, moved from loop to hook)
def permission_hook(block):
    if block.name == "bash":
        for pattern in DENY_LIST:
            if pattern in block.input.get("command", ""):
                return "Permission denied by deny list"
    if block.name in ("write_file", "edit_file"):
        path = block.input.get("path", "")
        if not (WORKDIR / path).resolve().is_relative_to(WORKDIR):
            choice = input("   Allow? [y/N] ").strip().lower()
            if choice not in ("y", "yes"):
                return "Permission denied by user"
    return None

# PreToolUse: logging
def log_hook(block):
    print(f"[HOOK] {block.name}(...)")

# PostToolUse: large output reminder
def large_output_hook(block, output):
    if len(str(output)) > 100000:
        print(f"[HOOK] ⚠ Large output from {block.name}")

register_hook("PreToolUse", permission_hook)
register_hook("PreToolUse", log_hook)
register_hook("PostToolUse", large_output_hook)
```

**Stop**, loop가 막 종료되려 할 때(`stop_reason != "tool_use"`) 트리거된다. 교육용 버전은 정리 요약을 출력한다:

```python
def summary_hook(messages: list) -> str | None:
    """Print a summary when the loop is about to stop."""
    tool_count = sum(1 for m in messages
                     for b in (m.get("content") if isinstance(m.get("content"), list) else [])
                     if isinstance(b, dict) and b.get("type") == "tool_result")
    print(f"\033[90m[HOOK] Stop: session used {tool_count} tool calls\033[0m")
    return None   # return None = allow stop, return string = force continuation

register_hook("Stop", summary_hook)
```

agent_loop에서는 종료 직전에 트리거된다:

```python
if response.stop_reason != "tool_use":
    force = trigger_hooks("Stop", messages)   # ← before exiting
    if force:
        # hook returned a message → inject it and continue
        messages.append({"role": "user", "content": force})
        continue
    return
```

**loop의 변경은 단 하나뿐**: s03은 `check_permission(block)`을 직접 호출했지만, s04는 이를 `trigger_hooks("PreToolUse", block)`으로 대체한다:

```python
for block in response.content:
    if block.type != "tool_use":
        continue

    # s03: if not check_permission(block): ...
    # s04: hooks replace hardcoding
    blocked = trigger_hooks("PreToolUse", block)
    if blocked:
        results.append({"type": "tool_result", "tool_use_id": block.id,
                        "content": str(blocked)})
        continue

    handler = TOOL_HANDLERS.get(block.name)
    output = handler(**block.input) if handler else f"Unknown: {block.name}"

    trigger_hooks("PostToolUse", block, output)

    results.append({"type": "tool_result", "tool_use_id": block.id,
                    "content": output})
```

네 개의 hook이 agent 사이클의 핵심 노드를 커버한다: 입력 → 실행 전 → 실행 후 → 종료. loop는 trigger_hooks()만 호출하고, 모든 로직은 hook 콜백 안에 산다.

---

## s03과의 변경점

| 구성 요소 | 이전 (s03) | 이후 (s04) |
|-----------|-------------|-------------|
| 확장 방식 | check_permission()을 loop에 하드코딩 | HOOKS registry + trigger_hooks() |
| 새 함수 | — | register_hook, trigger_hooks |
| Hook 콜백 | — | context_inject_hook, permission_hook, log_hook, large_output_hook, summary_hook |
| Loop | check_permission()을 직접 호출 | trigger_hooks("PreToolUse", ...) 호출 |
| 종료 제어 | 없음 | trigger_hooks("Stop", ...)가 종료를 막을 수 있음 |
| 입력 가로채기 | 없음 | trigger_hooks("UserPromptSubmit", ...)가 context를 주입할 수 있음 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s04_hooks/code.py
```

다음 prompt들을 시도해보라:

1. `Read the file README.md` (그대로 통과해야 하며, hook 로그를 관찰)
2. `Create a file called test.txt` (생성 후 PostToolUse가 발화하는지 관찰)
3. `Delete all temporary files in /tmp` (bash + rm이 권한 hook을 트리거)

관찰 포인트: 각 tool 실행 전에 `[HOOK]` 로그가 나타나는가? 권한이 거부될 때, 그것은 hook에 의해 가로채진 것인가, 아니면 loop에 하드코딩된 것인가?

---

## 다음 단계

이제 Agent는 작업을 안전하게 실행할 수 있다. 하지만 Agent가 "먼저 무엇을 하고, 다음에 무엇을 할까?"를 멈춰서 고민하기는 할까? 복잡한 작업이 주어졌을 때, 곧장 뛰어드는가, 아니면 먼저 계획하는가?

→ s05 TodoWrite: Agent에게 계획 도구를 주자. 먼저 리스트를 만들고, 그 다음에 실행하라.

<details>
<summary>CC 소스 코드 깊이 들여다보기</summary>

> 아래 내용은 CC 소스 코드 `toolHooks.ts`(650줄), `hooks.ts`, `stopHooks.ts`, `coreTypes.ts`에 대한 완전한 분석에 기반한다.

### 1. Hook 이벤트: 4개가 아니라 27개

교육용 버전은 PreToolUse와 PostToolUse만 다룬다. CC는 실제로 27개의 hook 이벤트를 가진다(`coreTypes.ts:25-53`):

| 분류 | 이벤트 |
|----------|--------|
| Tool 관련 | `PreToolUse`, `PostToolUse`, `PostToolUseFailure` |
| 세션 관련 | `SessionStart`, `SessionEnd`, `Stop`, `StopFailure`, `Setup` |
| 사용자 상호작용 | `UserPromptSubmit`, `Notification`, `PermissionRequest`, `PermissionDenied` |
| 서브 에이전트 | `SubagentStart`, `SubagentStop` |
| Compaction 관련 | `PreCompact`, `PostCompact` |
| 팀 관련 | `TeammateIdle`, `TaskCreated`, `TaskCompleted` |
| 기타 | `Elicitation`, `ElicitationResult`, `ConfigChange`, `WorktreeCreate`, `WorktreeRemove`, `InstructionsLoaded`, `CwdChanged`, `FileChanged` |

교육용 버전이 핵심 이벤트 4개(UserPromptSubmit, PreToolUse, PostToolUse, Stop)만 다루는 이유는, 이들이 완전한 agent 사이클의 모든 핵심 노드를 커버하기 때문이다. 나머지 23개도 동일한 패턴을 따른다.

### 2. HookResult 공통 필드

CC의 `HookResult`(`types/hooks.ts:260-275`)는 14개의 필드를 가진다. 자주 쓰이는 것들:

| 필드 | 타입 | 용도 |
|-------|------|---------|
| `message` | Message | 선택적 UI 메시지 |
| `blockingError` | HookBlockingError | 차단 에러 → 대화에 주입되어 모델이 스스로 교정 |
| `outcome` | success/blocking/non_blocking_error/cancelled | 실행 결과 |
| `preventContinuation` | boolean | 후속 실행 방지 |
| `stopReason` | string | 종료 사유 설명 |
| `permissionBehavior` | allow/deny/ask/passthrough | hook이 권한 결정을 반환 |
| `updatedInput` | Record | tool 입력 수정 |
| `additionalContext` | string | 추가 context |
| `updatedMCPToolOutput` | unknown | MCP tool 출력 수정 |

### 3. 핵심 불변식: Hook의 'allow'는 deny/ask 규칙을 우회할 수 없다

이것은 CC 권한 시스템에서 가장 중요한 보안 설계다(`toolHooks.ts:325-331`): **hook이 allow를 반환하더라도, 여전히 settings.json의 deny/ask 규칙을 체크한다.** 사용자의 hook 스크립트가 "allow"라고 말하더라도, 해당 tool이 settings.json에서 비활성화되어 있으면 그 동작은 여전히 차단된다.

교육용 버전에는 이 계층이 없다. hook이 non-None을 반환하면 곧바로 중단된다. 교육 목적으로는 충분하지만, 프로덕션에서라면 보안 취약점을 만들 것이다.

### 4. stopHookActive 메커니즘

CC의 Stop hook에는 무한 루프 방지 메커니즘이 있다(`query.ts:212,1300`): `stopHookActive` 상태 필드. stop hook이 blockingError를 만들면, loop는 `stopHookActive: true` 상태로 다시 진입한다. 이후 반복에서는 이 플래그를 보고 stop hook을 다시 트리거하지 않는다. 이는 결코 멈추지 않는 버그를 방지한다: 모델이 스스로 교정 → stop hook이 다시 에러 → 모델이 다시 교정 → stop hook이 다시 에러...

### 5. hook_stopped_continuation

PostToolUse hook이 `preventContinuation: true`를 반환하면, `hook_stopped_continuation` attachment가 생성된다(`toolHooks.ts:117-130`). query.ts(L1388-1393)가 이를 감지해 `shouldPreventContinuation = true`로 설정하고, loop가 종료되도록 한다. 이것이 "hook이 Agent를 우아하게 종료시키는" 메커니즘이다 — 크래시가 아니라 완료다.

### 교육용 버전의 단순화는 의도된 것이다

- 27개 이벤트 → 4개(UserPromptSubmit/PreToolUse/PostToolUse/Stop): agent 사이클의 핵심 노드를 커버
- 14개 필드 → 단순한 반환값(None = 계속, non-None = 중단/계속): 인지 부하 최소화
- Hook의 allow vs deny/ask 불변식 → 생략: 교육용 버전에는 settings.json 계층이 없음
- stopHookActive → 생략: 교육용 버전의 Stop hook은 단순한 continuation만 하므로 무한 루프 방지가 필요 없음

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
