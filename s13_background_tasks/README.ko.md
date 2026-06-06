# s13: Background Tasks — 느린 작업은 백그라운드로

[中文](README.md) · [English](README.en.md) · [日本語](README.ja.md) · [한국어](README.ko.md)

s01 → ... → s11 → s12 → `s13` → [s14](../s14_cron_scheduler/) → s15 → ... → s20

> *"느린 작업은 백그라운드로 보내고, agent는 계속 처리를 이어간다"* — 백그라운드 스레드가 명령을 실행하고, 완료되면 notification을 주입한다.
>
> **Harness Layer**: Background — 비동기 실행으로 메인 loop를 막지 않는다.

---

## 문제

세탁기를 써본 적 있는가? 옷을 넣고 시작 버튼을 누른 다음, 다른 일을 하러 간다 — 요리하고, 메시지에 답장하고, 논문을 읽는다. 30분 뒤 세탁기가 삑 소리를 낸다: 완료. 30분 동안 그 앞에 서서 기다리지 않는다.

agent의 bash tool도 마찬가지다. `pip install torch`는 10분, `npm run build`는 3분이 걸린다. 이 명령들이 실행되는 동안 agent는 bash가 반환되기를 기다리며, 그 시간을 다른 작업을 처리하는 데 쓰지 못한다.

파일을 읽는 건 밀리초 단위라 기다릴 필요가 없다. `git status`는 1초 안에 반환되니 기다릴 필요가 없다. 하지만 `npm install`은? 몇 분이 걸린다. agent는 10분 동안 아무것도 하지 않고 기다리는데, LLM 호출은 토큰 단위로 과금되므로 — 유휴 시간은 낭비다.

---

## 해결책

![백그라운드 작업 개요](images/background-tasks-overview.en.svg)

교육용 코드는 S12의 간소화된 task 시스템과 prompt 조립을 이어받았다. 백그라운드 작업에 집중하기 위해 완전한 오류 복구, memory, skill 시스템은 생략했다. 유일한 변경점은 다음과 같다: 느린 작업은 백그라운드 스레드로 보내고, agent는 loop를 계속 돌리며, 백그라운드 결과는 notification으로 주입된다.

동기(Sync) vs 백그라운드(Background):

| | 동기 (s12) | 백그라운드 (s13) |
|---|---|---|
| 느린 작업 | agent가 대기 | 백그라운드 스레드가 실행 |
| agent 유휴 | 있음 | 없음, 계속 처리 |
| 결과 | 즉시 반환 | 다음 turn에 notification 주입 |
| 판단 기준 | — | `run_in_background` 파라미터(모델의 명시적 요청), 휴리스틱 폴백 |

---

## 동작 방식

### should_run_background: 명시적 요청 우선, 휴리스틱 폴백

모델은 bash tool의 `run_in_background` 파라미터를 통해 백그라운드 실행을 명시적으로 요청한다. 모델이 지정하지 않으면, 교육용 버전은 키워드 휴리스틱으로 폴백한다:

```python
def is_slow_operation(tool_name: str, tool_input: dict) -> bool:
    """Fallback heuristic: commands likely to take > 30s."""
    if tool_name != "bash":
        return False
    cmd = tool_input.get("command", "").lower()
    slow_keywords = ["install", "build", "test", "deploy", "compile",
                     "docker build", "pip install", "npm install",
                     "cargo build", "pytest", "make"]
    return any(kw in cmd for kw in slow_keywords)

def should_run_background(tool_name: str, tool_input: dict) -> bool:
    """Model explicit request takes priority; fallback to heuristic."""
    if tool_input.get("run_in_background"):
        return True
    return is_slow_operation(tool_name, tool_input)
```

CC의 bash tool 스키마에는 `run_in_background: boolean` 파라미터가 있다(`BashTool.tsx:241`). 모델이 어떤 명령을 백그라운드로 보낼지 결정하므로 키워드 추측이 필요 없다. 교육용 버전은 휴리스틱을 폴백으로 유지하지만, 주요 경로는 모델의 명시적 요청이다.

### start_background_task: 백그라운드 실행과 라이프사이클

tool 호출을 worker 함수로 감싸 daemon 스레드로 디스패치한다. 각 백그라운드 task는 고유 ID를 받고, 상태는 `background_tasks` dict에서 추적된다:

```python
_bg_counter = 0
background_tasks: dict[str, dict] = {}   # bg_id → {tool_use_id, command, status}
background_results: dict[str, str] = {}   # bg_id → output
background_lock = threading.Lock()

def start_background_task(block) -> str:
    """Run tool in a daemon thread. Returns background task ID."""
    global _bg_counter
    _bg_counter += 1
    bg_id = f"bg_{_bg_counter:04d}"

    def worker():
        result = execute_tool(block)
        with background_lock:
            background_tasks[bg_id]["status"] = "completed"
            background_results[bg_id] = result

    with background_lock:
        background_tasks[bg_id] = {
            "tool_use_id": block.id,
            "command": block.input.get("command", ""),
            "status": "running",
        }
    thread = threading.Thread(target=worker, daemon=True)
    thread.start()
    return bg_id
```

단순한 `[Running in background...]` 대신 `bg_id`를 반환한다. `daemon=True`는 agent 프로세스가 종료될 때 스레드도 함께 종료되도록 보장한다. 교육용 버전은 추적에 인메모리 dict를 사용하지만, 실제 CC는 `LocalShellTaskState`를 두고 출력을 파일로 리디렉션하며, task 중지와 후속 출력 읽기까지 포함한 완전한 라이프사이클을 갖춘다.

### collect_background_results: notification 수집

백그라운드 task가 완료되면 결과를 수집해 `<task_notification>` 메시지로 포맷한다:

```python
def collect_background_results() -> list[str]:
    """Collect completed results as task_notification messages."""
    with background_lock:
        ready_ids = [bid for bid, task in background_tasks.items()
                     if task["status"] == "completed"]
    notifications = []
    for bg_id in ready_ids:
        with background_lock:
            task = background_tasks.pop(bg_id)
            output = background_results.pop(bg_id, "")
        notifications.append(
            f"<task_notification>\n"
            f"  <task_id>{bg_id}</task_id>\n"
            f"  <status>completed</status>\n"
            f"  <command>{task['command']}</command>\n"
            f"  <summary>{output[:200]}</summary>\n"
            f"</task_notification>")
    return notifications
```

notification은 원래의 `tool_use_id`를 재사용하지 않는다. 원래의 tool 호출은 이미 placeholder `tool_result`로 응답이 끝났다. 백그라운드 완료는 독립적인 이벤트이므로 `task_notification` 형식으로 주입된다. 이는 Messages API의 tool 페어링 규칙을 존중한다: 하나의 `tool_use`에는 정확히 하나의 `tool_result`가 대응된다.

### Loop 통합

agent loop에서 tool 실행은 두 경로로 분기된다. notification과 결과는 하나의 user 메시지로 병합된다:

```python
results = []
for block in response.content:
    if block.type != "tool_use":
        continue
    if should_run_background(block.name, block.input):
        bg_id = start_background_task(block)
        results.append({"type": "tool_result",
            "tool_use_id": block.id,
            "content": f"[Background task {bg_id} started] "
                       f"Result will be available when complete."})
    else:
        output = execute_tool(block)
        results.append({"type": "tool_result",
            "tool_use_id": block.id, "content": output})

# Merge notifications and tool results into one user message
user_content = []
bg_notifications = collect_background_results()
if bg_notifications:
    for notif in bg_notifications:
        user_content.append({"type": "text", "text": notif})
user_content.extend(results)
messages.append({"role": "user", "content": user_content})
```

느린 작업은 `bg_id`가 담긴 placeholder tool_result를 받는다. 덕분에 LLM은 이 명령이 아직 실행 중임을 알고 먼저 다른 일을 할 수 있다. 백그라운드가 완료되면, 해당 notification은 현재 turn의 tool_result들과 함께 독립적인 text block으로 하나의 user 메시지에 주입된다.

교육용 버전은 agent loop가 계속 도는 동안 백그라운드 결과를 폴링한다. 실제 CC는 notification 큐(`messageQueueManager.ts`)를 사용해 백그라운드 완료 이벤트를 후속 turn에 전달하며, tool loop를 기다리지 않는다.

### 종합

```
Turn 1:
  LLM → bash "npm install" (run_in_background=true)
  → start_background_task → bg_0001
  → tool_result: "[Background task bg_0001 started]..."
  → LLM: "OK, I'll check later. Let me also read the config."

Turn 2:
  LLM → read_file "package.json" (fast, sync)
  → tool_result: file content
  → collect: bg_0001 done! inject <task_notification>
  → LLM sees: config file + install notification in one message
```

agent는 기다리지 않았다 — npm install이 백그라운드에서 도는 동안, config 파일을 읽었다.

---

## s12에서 달라진 점

| 구성 요소 | 이전 (s12) | 이후 (s13) |
|-----------|-------------|-------------|
| 실행 모델 | 모두 동기 | 느린 작업은 백그라운드 스레드 + notification 주입 |
| bash 스키마 | `command` | `command` + `run_in_background` |
| 새 함수 | — | `should_run_background`, `is_slow_operation`, `start_background_task`, `collect_background_results` |
| 새 타입 | — | `background_tasks: dict`, `background_results: dict`, `background_lock: Lock` |
| notification 형식 | — | `<task_notification>` (tool_use_id 재사용 안 함) |
| Loop 동작 | tool이 순차 실행 | 느린 작업은 비동기, 빠른 작업은 동기, 매 turn마다 notification 수집 |
| Tools | 8개 (s12) | 8개 (변동 없음, 실행 전략만 변경) |

---

## 직접 해보기

```sh
cd learn-claude-code
python s13_background_tasks/code.py
```

다음 prompt들을 시도해보자:

1. `Run pip list in the background and find all Python files in this directory`
2. `Run npm install (use run_in_background) and while waiting, read package.json`
3. `Create a task to setup the project, then run pip list in the background`

관찰할 점: 느린 작업이 백그라운드로 디스패치되는가? `bg_id`가 반환되는가? 백그라운드 notification이 `<task_notification>` 형식으로 주입되는가?

---

## 다음 단계

백그라운드 작업은 "느린 작업이 막지 않게 하기" 문제를 해결했다. 하지만 정해진 일정에 따라 무언가를 하고 싶다면? 예를 들어 "매일 아침 9시에 테스트 실행"이나 "5분마다 서버 상태 확인" 같은 것 말이다.

s14 Cron Scheduler → agent에게 알람 시계를 주자.

<details>
<summary>CC 소스 코드 깊이 파보기</summary>

> 아래 내용은 CC 소스 코드 `query.ts`(라인 211, 1054-1060, 1411-1482), `services/toolUseSummary/toolUseSummaryGenerator.ts`(L15 prompt 텍스트), `LocalShellTask.tsx`(L24-25 상수, L59-98 watchdog 로직), `messageQueueManager.ts`(notification 큐), `utils/task/framework.ts`(L267 `enqueueTaskNotification`)를 기반으로 한 완전한 분석이다.

### 1. pendingToolUseSummary: Haiku 백그라운드 생성

CC는 매 tool 실행 배치가 끝날 때마다 Haiku 사이드 쿼리를 시작해 tool use 요약을 생성한다. `query.ts:1411-1482`에서 시작되며, prompt 텍스트는 `services/toolUseSummary/toolUseSummaryGenerator.ts:15`(변수 `TOOL_USE_SUMMARY_SYSTEM_PROMPT`)에 정의되어 있다. prompt는 "Write a short summary label... think git-commit-subject, not sentence", 즉 과거형, 약 30자다.

Haiku 요약(~1초)은 메인 모델의 스트리밍 출력(5-30초) 동안 완료된다. 다음 turn이 시작되기 전에 요약이 yield된다. SDK consumer는 이 요약을 모바일 진행 표시에 활용한다.

### 2. 스레드 모델: 진짜 스레드는 없다

CC는 Node.js/Bun의 단일 스레드 이벤트 루프 위에서 동작한다. "백그라운드"는 단지 "await하지 않는다"를 의미할 뿐이다. `ShellCommand.background(taskId)`는 stdout/stderr를 파일로 리디렉션해 프로세스가 독립적으로 실행되게 한다.

### 3. 일곱 가지 백그라운드 task 타입

CC는 7가지 백그라운드 task 타입을 정의한다(`Task.ts:7-13`): `local_bash`, `local_agent`, `remote_agent`, `in_process_teammate`, `local_workflow`, `monitor_mcp`, `dream`. 각각은 고유한 등록, 라이프사이클, notification 메커니즘을 갖는다.

### 4. notification 주입: 커맨드 큐

백그라운드 task가 완료되면 `enqueueTaskNotification`(`utils/task/framework.ts:267`) 또는 `enqueuePendingNotification`(`messageQueueManager.ts`)을 통해 공유 커맨드 큐에 인큐된다. notification 형식은 구조화된 XML이다:

```xml
<task_notification>
  <status>completed</status>
  <summary>Background command "npm test" completed (exit code 0)</summary>
</task_notification>
```

우선순위는 `next` > `later`다(`messageQueueManager.ts`). 백그라운드 task는 기본적으로 `later`다(user 입력을 막지 않음). 소비 지점은 `query.ts:1566-1593`이다.

### 5. 정체(Stall) Watchdog

백그라운드 bash task에는 watchdog가 있어(`LocalShellTask.tsx` L24-25 상수, L59-98 로직) 출력이 정체되었는지 주기적으로 확인한다. 45초 동안 출력이 늘지 않으면 인터랙티브 prompt(`(y/n)` 등)를 감지해, 백그라운드 task가 응답되지 않은 인터랙티브 대화창에 걸려 멈추는 것을 방지한다.

### 6. 동시성 제한

포그라운드 tool 호출: `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`(기본값은 안전한 tool 10개 동시 실행). 백그라운드 bash task: 하드 리밋 없음, 독립적인 서브프로세스다.

</details>

<!-- translation-sync: zh@v1, en@v1, ja@v1 -->
