# s05: TodoWrite — 계획 없는 Agent는 길을 잃는다

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → s02 → s03 → s04 → `s05` → [s06](../s06_subagent/) → s07 → ... → s20

> *"계획 없는 Agent는 바람 부는 대로 흘러간다"* — 먼저 단계를 나열하고, 그다음 실행하라. 복잡한 작업에서 단계를 놓칠 확률이 줄어든다.
>
> **Harness 레이어**: Planning — Agent가 행동하기 전에 먼저 생각하게 한다.

---

## 문제

Agent에게 복잡한 작업을 준다: "모든 Python 파일을 snake_case로 이름을 바꾸고, 테스트를 실행하고, 실패를 고쳐라."

Agent가 작업을 시작해 파일 3개의 이름을 바꾸고, 테스트를 실행하고, 실패 2개를 발견하고, 고치기 시작한다. 고치는 동안 원래 목표가 "snake_case로 이름 바꾸기"였다는 사실을 잊어버린다. 테스트 실패가 주의력을 모두 빨아들인 것이다.

대화가 길어질수록 상황은 더 나빠진다. tool 결과가 계속 context를 채우면서 system prompt의 영향력을 희석시킨다. 10단계짜리 리팩터링에서 1~3단계가 끝나고 나면 4~10단계는 이미 Agent의 주의에서 밀려났기 때문에, Agent는 즉흥적으로 행동하기 시작한다.

---

## 해결책

![Todo 개요](images/todo-overview.en.svg)

이전 챕터의 최소 hook 구조는 그대로 유지하고, 새로운 `todo_write` tool과 reminder 메커니즘에 집중한다. `todo_write`는 실제 작업을 하지 않는다. 파일을 읽거나 명령을 실행할 수 없으며, 단지 Agent가 본격적으로 뛰어들기 전에 생각을 정리하게 해줄 뿐이다.

dispatch 메커니즘은 바뀌지 않았다. 새 tool도 여전히 `TOOL_HANDLERS[block.name]`을 통해 라우팅된다. 다만 todo reminder를 보여주기 위해 loop에 카운터를 하나 추가했다. `todo_write`를 호출하지 않은 라운드가 3번 연속되면 reminder가 주입된다.

---

## 동작 방식

**todo_write tool**은 상태(status)가 포함된 리스트를 받아 현재 프로세스 메모리에 보관하고, 진행 상황을 터미널에 표시한다:

```python
CURRENT_TODOS: list[dict] = []

def run_todo_write(todos: list) -> str:
    global CURRENT_TODOS
    CURRENT_TODOS = todos

    lines = ["\n## Current Tasks"]
    for t in CURRENT_TODOS:
        icon = {"pending": " ", "in_progress": "▸", "completed": "✓"}[t["status"]]
        lines.append(f"  [{icon}] {t['content']}")
    print("\n".join(lines))
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

tool 정의는 dispatch map에 있는 나머지 5개에 합류한다:

```python
TOOLS = [
    {"name": "bash",       ...},
    {"name": "read_file",  ...},
    {"name": "write_file", ...},
    {"name": "edit_file",  ...},
    {"name": "glob",       ...},
    # s05: new entry
    {"name": "todo_write", "description": "Create and manage a task list ...",
     "input_schema": {
         "type": "object",
         "properties": {
             "todos": {
                 "type": "array",
                 "items": {
                     "type": "object",
                     "properties": {
                         "content": {"type": "string"},
                         "status": {"type": "string", "enum": ["pending", "in_progress", "completed"]},
                     },
                 },
             },
         },
     },
    },
]

TOOL_HANDLERS["todo_write"] = run_todo_write
```

**Nag reminder**: 모델이 3라운드 연속으로 `todo_write`를 호출하지 않으면 reminder가 자동으로 주입된다(교육용 메커니즘이며, CC 소스에는 고정된 라운드 카운트 로직이 없다):

```python
if rounds_since_todo >= 3 and messages:
    messages.append({
        "role": "user",
        "content": "<reminder>Update your todos.</reminder>",
    })
    rounds_since_todo = 0
```

Agent가 작업을 받았을 때의 전형적인 흐름: 먼저 `todo_write`를 호출해 모든 단계를 나열하고(전부 `pending`) → 한 단계를 골라 `in_progress`로 설정 → 완료하면 `completed`로 설정 → 다음 `pending`을 보고 → 계속 진행. `todo_write` 없이 3라운드가 지나면 loop는 다음 LLM 호출 전에 reminder를 덧붙인다.

**핵심 통찰**: todo_write는 Agent에게 추가적인 **실행 능력**을 주지 않는다. 그것이 더해주는 것은 **계획 능력**이다.

---

## s04에서 바뀐 점

| 구성 요소 | 이전 (s04) | 이후 (s05) |
|-----------|-------------|-------------|
| Tool 개수 | 5개 (bash, read, write, edit, glob) | 6개 (+todo_write) |
| Planning | 없음 | 상태를 가진 TODO 리스트 + nag reminder |
| SYSTEM prompt | 범용 prompt | "실행 전에 계획하라" 가이드 추가 |
| Loop | 변경 없음 | dispatch는 그대로, rounds_since_todo 카운터와 reminder 주입 추가 |

---

## 직접 해보기

```sh
cd learn-claude-code
python s05_todo_write/code.py
```

다음 prompt들을 시도해 보라:

1. `Refactor s05_todo_write/example/hello.py: add type hints, docstrings, and a main guard` (먼저 3단계를 나열한 뒤 실행해야 한다)
2. `Create a Python package under s05_todo_write/example/demo_pkg with __init__.py, utils.py, and tests/test_utils.py`
3. `Review Python files under s05_todo_write/example and fix any style issues`

관찰 포인트: 첫 tool 호출이 `todo_write`였는가? 몇 개의 TODO 단계가 나열되었는가? 실행 중에 status가 `pending`에서 `in_progress` / `completed`로 옮겨갔는가?

---

## 다음 단계

이제 Agent는 계획을 세울 수 있다. 하지만 작업이 너무 크면, 예를 들어 "auth 모듈 전체를 리팩터링하라" 같은 경우, TODO 리스트만으로는 충분하지 않다. 그 작업 자체가 수십 개의 하위 작업의 집합이며, 단일 대화의 context 안에서는 익사해 버린다.

→ s06 Subagent: 큰 작업을 하위 작업으로 쪼개고, 각각을 자신만의 깨끗한 context를 가진 독립적인 Agent가 처리하게 한다. 서로 오염되지 않는다.

<details>
<summary>CC 소스 코드 파고들기</summary>

CC에는 두 개의 task 시스템이 공존한다 (`tasks.ts:133-139`):

- **TodoWrite (V1)**: 단순한 리스트 tool로, 데이터는 메모리상의 AppState에 유지된다 (`TodoWriteTool.ts:65-103`). 교육용 버전도 마찬가지로 프로세스 메모리에 보관하고 종료 시 비운다.
- **Task System (V2 = s12)**: 파일로 영속화되며, 의존성 그래프, 동시성 락, 소유권(ownership)을 가진다.

전환은 `isTodoV2Enabled()`로 제어된다. 현재 소스 기준: V2는 인터랙티브 세션에서 기본으로 활성화되고, V1은 비인터랙티브(SDK) 세션에서 사용된다. `CLAUDE_CODE_ENABLE_TASKS`를 설정하면 무조건 V2가 강제된다. 소스의 "Force-enable tasks in non-interactive mode" 주석은 env var 경로의 목적을 설명하는 것이지, 기본 분기의 반환 의미를 설명하는 것이 아니라는 점에 유의하라.

교육용 버전은 실제 소스의 `activeForm` 필드를 생략했다 (`utils/todo/types.ts:8-15`). CC는 이 필드를 UI 스피너에서 "지금 무엇을 하고 있는지" 표시하는 데 사용하지만, 교육용 버전은 터미널 출력만 있으므로 이 필드가 필요 없다.

교육용 버전의 nag reminder(업데이트 없이 3라운드가 지나면 주입을 트리거)는 교육용 메커니즘이다. CC 소스에는 고정된 "3라운드" 로직이 없다. 가장 가까운 것은 `TodoWriteTool.ts:72-107`로, 3개 이상의 todo가 모두 completed인데 verification 항목이 없을 때 verification nudge를 덧붙인다.

Task System이 TodoWrite보다 핵심적으로 더 가진 점:
- 메모리 리스트 대신 파일 영속화 (Claude 설정 디렉터리 `tasks/{taskListId}/{taskId}.json`)
- 평면 리스트 대신 `blockedBy` 의존성 그래프
- 락 없음 대신 `proper-lockfile` 동시성 안전성
- 하나의 tool 대신 4개의 분리된 tool (Create/Get/Update/List)
- 외부 시스템 통합을 위한 TaskCreated / TaskCompleted hook (`TaskCreateTool.ts:80-129`, `TaskUpdateTool.ts:231-260`)

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
